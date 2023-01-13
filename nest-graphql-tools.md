# Nest Graphql Tools Document

2023-01-13 초안 작성

## [모듈] GqlGatewayModule

---

- Apollo 서버 모듈
- 옵션은 nestjs 의 [ApolloDriverOption](https://docs.nestjs.com/graphql/quick-start#example) + graphql-shield의 [Shield Options](https://the-guild.dev/graphql/shield/docs/shield) 로 구성
- Nestjs 의 Schema First 방식과 호환

#### GqlGatewayModule.forRoot()

```ts
@Module({
  imports: [
    GqlGatewayModule.forRoot({
      installSubscriptionHandlers: true,
      subscriptions: {
        "graphql-ws": false,
        "subscriptions-transport-ws": true,
      },
      shieldOptions: { debug: true },
    }),
  ],
})
export class AppModule {}
```

#### GqlGatewayModule.forRootAsync()

```ts
@Module({
  imports: [
    GqlGatewayModule.forRootAsync({
      imports: [],
      inject: [],
      useFactory(...args) {
        return {
          installSubscriptionHandlers: true,
          subscriptions: {
            "graphql-ws": false,
            "subscriptions-transport-ws": true,
          },
          shieldOptions: { debug: true },
        };
      },
    }),
  ],
})
export class AppModule {}
```

<br/>
<br/>

## [모듈] RemoteSchemaModule

---

- 원격 스키마 연결

### 옵션

```ts
export interface RemoteGqlSchemaOptions {
  uri: string; // 연결할 원격 스키마 (Introspection 을 지원해야함)
  name: string; // 스키마 명 지정 (추후 Delegate시 필요) // 추후 개선버전에선 삭제 예정
  headers?: Record<string, any>; //Introspection Query시 필요한 헤더
  batch?: boolean; // 여러 쿼리를 묶어 보낼지 여부
  subscriptionProtocol?: "ws" | "legacy"; // Subscription시 // ws : graphql-ws | legacy : graphql-transport-ws
}
```

### 예제

```ts
@Module({
  imports: [
    GqlGatewayModule.forRootAsync({
      imports: [],
      inject: [],
      useFactory(...args) {
        return {
          installSubscriptionHandlers: true,
          subscriptions: {
            "graphql-ws": false,
            "subscriptions-transport-ws": true,
          },
          shieldOptions: { debug: true },
        };
      },
    }),
    RemoteSchemaModule.connectGql({
      subscriptionProtocol: "ws",
      batch: true,
      name: "UserSchema",
      uri: "http://localhost:5204/v1/graphql",
      headers: {
        "x-hasura-admin-secret": "msa-test",
      },
    }),
  ],
})
export class AppModule {}
```

<br/>
<br/>

## [클래스 데코레이터] @GatewayResolver

---

- @nestjs/graphql 의 Resolve 에 상응
- 원격 스키마 제어에 필요한 데코레이터 및 메서드 등록
- @nestjs/graphql 의 Query, Mutation, Subscription 호환

```ts
type GatewayResolver = (typeRoot?: string) => ClassDecorator;
```

```ts
import { Query } from "@nestjs/graphql";

@GatewayResolver()
export class MyGatewayResolver {
  @Query()
  sayHello() {
    return "Hello World!";
  }
}
```

@nestjs/graphql의 Schema First 방식으로 작성<br/>
따라서 대부분의 경우 따로 graphql typedef 를 명시해줘야 하기에 TypeDef 데코레이터와 함께 사용

## [클래스 데코레이터] @TypeDef

---

- 서버 전체의 Graphql Schema 에 추가되는 type def 정의를 위한 데코레이터
- 중복되는 타입은 자동으로 머지되며 이걸 이용해 여러 원격 스키마간 조인도 가능하다
- graphql-tag 라이브러리를 사용하면 IDE의 코드 하이라이팅, 정렬 기능도 사용 가능하다.

```ts
type TypeDef = (...types: Array<DocumentNode | string>) => ClassDecorator;
```

```ts
import gql from "graphql-tag";
import { Args, Query } from "@nestjs/graphql";

@GatewayResolver()
@TypeDef(gql`
  type Query {
    sayHello(yourName: String!): String!
  }
`)
export class MyGatewayResolver {
  @Query()
  sayHello(@Args("yourName") yourName: string) {
    return `Hello ${yourName}!`;
  }
}
```

@nestjs/graphql 의 [파라메터 데코레이터](https://docs.nestjs.com/graphql/resolvers#graphql-argument-decorators) @Args(), @Context(), @Info(), @Root() 모두 사용 가능하다

추가로 본 라이브러리에서 제공되는 @Delegate() 데코레이터는 커스텀 로직과 원격 스키마의 연결에 유용하다

```graphql
# 원격 스키마

type User {
  id: ID!
  email: String!
  password: String!
}

type Query {
  getUserById(id: ID!): User
}
```

```ts
@GatewayResolver()
@TypeDef(gql`
  type Query {
    whoAmI: User!
  }
`)
export class MyGatewayResolver {
  @Query()
  whoAmI(@Delegate() d: Delegator, @Context() ctx: any) {
    const userId = ctx.req.headers["x-user-id"];
    return d.delegateSingle({
      args: () => ({ id: userId }),
      fieldName: "getUserById",
      schemaKey: "UserSchema",
      operation: OperationTypeNode.QUERY,
    });
  }
}
```

이제 Playground 에서 아래와 같이 쿼리할수 있다

```graphql
query {
  whoAmI {
    id
    email
    password
  }
}
# Header : {
#   x-user-id: 1
# }
```

<br/>
<br/>

다음으론 ObjectType에 추가적인 필드를 더해주거나 기존 필드의 값을 변형하는 데코레이터에 대해 살펴본다.
<br/>

## [메서드 데코레이터] @FieldResolver

---

```ts
interface FieldResolverOptions {
  selectionSet?: string;
  comment?: string;
  directives?: string[];
  args?: Record<string, string>;
  extend?: boolean;
  resolveType?: (root: any) => string;
  nullable?: boolean;
}

type FieldResolver = (
  resolveTo: string,
  options?: FieldResolverOptions
) => MethodDecorator;
```

@FieldResolver 를 사용하기 위해선 @GatewayResolver에 확장/변경 하고자 하는 필드의 부모 타입명을 입력해준다.<br/>
따로 TypeDef는 입력하지 않아도 되지만 resolveTo에 해결되는? 리턴하는 Graphql 타입을 입력해준다.

```graphql
# graphql 예제 타입
type User {
  firstName: String!
  lastName: String!
  id: Int!
}

type Query {
  getUser: User
}
```

```ts
@GatewayResolver("User")
export class MyGatewayResolver {
  @FieldResolver("String", { selectionSet: `{ firstName lastName }` })
  async name(@Root() user: any) {
    return `${user.firstName} ${user.lastName}`;
  }
}
```

이제 Playground에서 아래와 같은 쿼리가 가능해진다

```graphql
query {
  getUser {
    firstName
    lastName
    name
  }
}
```

selectionSet 필드는 위 경우 필수이다 (name 필드를 resolve 하기위해 root 타입에서 firstName, lastName 필드가 필요하다고 명시하여 root 를 resolve 할때 해당 필드들이 요청되지 않았더라도 요청해 가지고 온다)
예로 위 예제와 비슷하지만 아래 쿼리는 작동하지 않는다!

```ts
  @FieldResolver("String")
  async name(@Root() user: any) {
    // 여기서 user = {id: 1} 이다 firstName 과 lastName이 존재하지 않으므로 빈 문자열이 리턴된다
    return `${user.firstName} ${user.lastName}`;
  }
```

```graphql
query {
  getUser {
    id
    name
  }
}
```

하지만 selectionSet을 정의했다면

```ts
  @FieldResolver("String", { selectionSet: `{ firstName lastName }` })
  async name(@Root() user: any) {
    // user = {id : 1, firstName : John, lastName: Doe }
    return `${user.firstName} ${user.lastName}`;
  }
```

쿼리에서 요청하지 않았더라도 해당 필드들이 붙어서 root 에 들어간다

## 원격 스키마 조인

TypeDef 의 스키마 머지를 @FieldResolver, Delegate과 함께 활용하면 두 원격스키마간 조인도 쉽게 가능하다

```graphq께
# 1번 서버/원격 스키마
type User {
  id: ID!
  email: String!
  password: String!
}

type Query {
  getUserById(id: ID!): User
  getUsersByIds(ids: [ID]): [User]
}

# 2번 서버/원격 스키마
type Order{
  id: ID!
  itemId: Int!
  price: Float!
  userId: Int!
}

type Query {
  ordersByUser(userId: Int!): [Order]
  ordersByItem(itemId: Int!): [Order]
}

```

```ts
@GatewayResolver("User")
export class MyUserResolver {
  @FieldResolver("[Order]", { selectionSet: `{ id }` })
  async orders(@Delegate() d: Delegator) {
    return d.delegateSingle({
      args(root) {
        return {
          id: root.id,
        };
      },
      fieldName: "ordersByUser",
      schemaKey: "orders",
      operation: OperationTypeNode.QUERY,
    });
  }
}
@GatewayResolver("Order")
export class MyGatewayResolver {
  @FieldResolver("User", { selectionSet: `{ userId }` })
  async user(@Delegate() d: Delegator) {
    return d.delegateBatch({
      keySelector(root) {
        return root.userId;
      },
      argsFromKeys(keys) {
        return {
          ids: keys,
        };
      },
      fieldName: "getUsersByIds",
      schemaKey: "users",
      operation: OperationTypeNode.QUERY,
    });
  }
}
```

아래 delegateBatch 는 root 가 배열로 반환될때 해당 배열의 길이만큼 Delegate 를 실행하는 비효율적인 N+1 쿼리를 방지한다<br/>
delegateBatch는 User 쪽 스키마에서 배열로 쿼리가 가능할때만 사용할 수 있다.<br/>
이제 아래 쿼리가 가능해진다

```graphql
query {
  getUserById(id: 1) {
    id
    orders {
      id
      price
    }
  }
  getOrdersByItem(itemId: 1) {
    id
    price
    user {
      id
      email
    }
  }
}
```


## [메서드 데코레이터] @Middleware
- 작성 예정
## [인터페이스] IGqlMiddleware
---
- 작성 예정
## [메서드 데코레이터] @Shield
- 작성 예정
## [인터페이스] IShieldResolver
- 작성 예정
---
## [메서드 데코레이터] @PassThrough
- 작성 예정
---
## [GatewayResolver 확장 Class] HasuraShieldRule
- 작성 예정
## [GatewayResolver 확장 Class] BaseShieldRule
- 작성 예정
---
## [클래스 데코레이터] GlobalMiddleware
- 작성 예정
---
## [클래스 데코레이터] InterfaceResolver
- 작성 예정
---
## [클래스 데코레이터] UnionResolver
- 작성 예정