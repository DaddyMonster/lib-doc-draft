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

- @Query, @Mutation, @Subscription 과 함께 쓰이는 미들웨어
- [graphql-middleware](https://github.com/dimatill/graphql-middleware) 패키지에 기반하였다.
- 아래 타입들로 구성된 Array를 인자로 받는다
  <br/>

```ts
type Middleware = (...args: Array<MiddlewareFunction | MiddlewareWithOptions | IGqlMiddleware> ) => MethodDecorator

type MiddlewareFunction (
      resolve: () => Promise<any>,
      parent: any,
      args: any,
      context: any,
      info: any
    ) => any

interface IMiddlewareWithOptions<TSource = any, TContext = any, TArgs = any> {
    fragment?: IMiddlewareFragment;
    fragments?: IMiddlewareFragment[];
    resolve?: IMiddlewareResolver<TSource, TContext, TArgs>;
}

abstract class IGqlMiddleware<
  TSource = any,
  TContext = any,
  TArgs = any
> {
  fragment?: string;
  abstract handleMiddleware(
    resolve: (
      source?: TSource,
      args?: TArgs,
      context?: TContext,
      info?: GraphQLResolveInfo & {
        stitchingInfo?: StitchingInfo;
      }
    ) => any,
    parent: TSource,
    args: TArgs,
    context: TContext,
    info: GraphQLResolveInfo
  ): Promise<any>;
}
```

일반적으로

1. root 타입에 기반한 미들웨어는 IMiddlewareWithOptions
2. root 타입과 관계 없는 미들웨어는 MiddlewareFunction
3. Nestjs Provider 의존성이 있는경우 IGqlMiddleware 를 implement 하는 클래스를 사용한다.<br/>

- 방식은 Fastfy 의 Middleware 같이 resolve 실행시 다시 미들웨어 혹은 리졸버를 실행한 값을 받아오며 값을 리턴시 해당 값으로 Resolve 된다
- target class에 @GatewaResolver() 데코레이터가 사용되어야만 적용된다.
- @nestjs/graphql 의 @Query | @Mutation | @Subscription 과 함께 사용할때만 적용된다.

```ts
// MiddlewareFunction
const loggingMiddleware: MiddlewareTypes = async (
  resolve,
  parent,
  args,
  context,
  info
) => {
  const start = Date.now();
  const resolveResult = await resolve(parent, args, context, info);
  const timeTaken = Date.now() - start;
  Logger.debug(`Query ${info.fieldName} took ${timeTaken}ms`);
  return resolveResult;
};
const logIdMiddleware: IMiddlewareWithOptions = {
  fragment: "fragment NodeId on Node { id }",
  resolve: async (resolve, parent, args, context, info) => {
    const result = await resolve(parent, args, context, info);
    Logger.debug(`Query ${info.fieldName} returned id ${result.id}`);
    return result;
  },
};

/* 
 아래 Class 방식이 가장 추천하는 방식이다
 DI가 가능하며
 @Resolve 데코레이터는 (MiddlewareResolver 타입의 함수) 
 리졸버에 요청된 정보를 변형하기 편하다
*/
@Injectable()
export class ModifyArgumentAndCheckInvitedMiddleware implements IGqlMiddleware {
  // Nestjs 의 DI 를 사용할 수 있다
  constructor(private db: DatabaseService) {}

  fragment = "fragment PostId on Post { id }";
  async handleMiddleware(@Resolve() resolve: MiddlewareResolver): Promise<any> {
    const isValid = await this.db.checkInvited();
    if (!isValid) return new Error("Not invited user");
    // 아래 쿼리의 limit 은 요청과 관계없이 10으로 고정된다
    return resolve({ args: { limit: 10 } });
  }
}

@GatewayResolver()
export class ExampleResolver {
  constructor(private postService: PostService) {}

  @Middleware(
    loggingMiddleware,
    logIdMiddleware,
    ModifyArgumentAndCheckInvitedMiddleware
  )
  @Query()
  getPosts(@Args("limit", { nullable: true }) limit: number) {
    // 위 적용된 미들웨어덕분에 limit Args가 클라이언트에서 누락하더라도 10으로 고정된다
    return this.postService.getPosts(limit);
  }
}
```

## [인터페이스] IGqlMiddleware

---

- 위 ModifyArgumentAndCheckInvitedMiddleware 참고

## [메서드 데코레이터] @Shield

- [graphql-shield](https://the-guild.dev/graphql/shield/docs) 기반의 rule을 인자로 받는다
- @nestjs/graphql 의 @Query @Mutation @Subscription 과 함께 사용 가능하다
- nest-graphql-tools 의 @FieldResolver @PassThrough 와 함께 사용 가능하다
- DI 가 필요한 경우 IShieldResolver 를 implement 한 클래스 작성하여 사용한다

```ts
import { allow, and, deny, inputRule, not, or, rule } from 'graphql-shield';

const isAdm1 = rule()(async (parent, args, ctx, info) => {
  return ctx.req.headers['x-auth'] === 'admin1';
});
const isAdm2 = rule()(async (parent, args, ctx, info) => {
  return ctx.req.headers['x-auth'] === 'admin2';
});
const isAdm3 = rule()(async (parent, args, ctx, info) => {
  return ctx.req.headers['x-auth'] === 'admin3';
});

const isAdm = or(isAdm1, isAdm2, isAdm3);
const isSuperAdm = and(isAdm, not(isAdm3));

const limitRule = inputRule()((yup, ctx) =>
  yup.object({
    limit: yup.number().min(1).max(10).default(5),
    offset: yup.number().required(),
  })
);

@GatewayResolver()
class SomeResolver {

  @Shield(and(or(isAdm, isAdm3), limitRule))
  @Query()
  async adm1and3AllowedQuery() {
    ...
  }
}
```

- IShieldResolver 를 Implement 하는 클래스를 정의해 사용도 가능하다

```ts
@Injectable()
export class ShieldService {
  async halfChance() {
    // 50% chance to return true
    const res = Math.random() > 0.5;
    console.log("SHILED CALLED", res ? "TRUE" : "FALSE");
    await sleep(1000);
    return res;
  }
}

import { Args, Root, Context, Info } from "@nestjs/graphql";
// @nestjs/graphql 의 Param Decorator 사용 가능
@Injectable()
export class HalfChanceShield implements IShieldResolver {
  constructor(private service: ShieldService) {}

  handleShieldRule(
    @Root() root: any
  ): boolean | Promise<boolean> | string | Promise<string> {
    return this.service.halfChance();
  }
}
```

하지만 Class 로 정의시 or and not 등의 graphql-shield에서 제공하는 오퍼레이터 사용이 불가능하다.

```ts
  // 아래 코드는 동작하지만 and or not 등의 오퍼레이터는 사용할 수 없다
  @Shield(HalfChanceShield)
  @Query()
  async halfChanceSuccessQuery() {
    ...
  }
```

아래와 같은 방법으로 클래스로 정의된 rule을 graphql-shield rule 형태로 만들수 있다
<br/>
이제 graphql-shield 의 오퍼레이터도 사용할 수 있다.

```ts
import { rule , not, and } from 'graphql-shield';
import { injectedRuleHandler } from 'nest-graphql-tools'
/*  Class Shield Rule은 injectedRuleHandler 를 사용해 변경하면
 DI 와 데코레이터의 이점은 누리면서 Shield Operator도 함께 사용 가능  */
const halfChangeRule = rule()(injectedRuleHandler(HalfChanceShield));

const quarterChangeRule = and(halfChangeRule, halfChangeRule)

  @Shield(not(quarterChangeRule))
  @Query()
  async halfAndQuarterChangeQuery() {
    ...
  }
```

## [인터페이스] IShieldResolver

- 위 예제 참고

---

## [메서드 데코레이터] @PassThrough

- PassThrough 는 원격 스키마로 Proxy 하여 Resolve 하는 쿼리에 대한 미들웨어이다

```graphql
# 원격스키마
type ChatMessage {
  id: ID!
  message: String!
  authorId: Int!
}
type Query {
  getChatMessages: [ChatMessage]
}
# --------
```

위와 같은 원격 스키마를 RemoteSchemaModule.connectGql 을 사용해 연결했을때
<br/>
별도의 코드 없이도 Playground 에서 작동하는하는 것으로 위에서 설명했다.

@PassThrough는 이러한 Proxy되는 쿼리들에 대한 제어를 위한 데코레이터이다.

- 첫번째 인자로 오퍼레이션 타입이 필요하다
- Field 단위에서도 사용이 가능하지만 @FieldResolver를 사용하는게 더 적절하다고 생각해 Type을 Query Mutation Subscription 으로만 한정했다. (Use Case가 있다면 타입 및 필드 단위의 미들웨어 및 쉴드로써 작성하게끔 바꾸는 것도 고려중이다.)
- @Shield 데코레이터와 함께 사용도 가능하다
- @Resolve 데코레이터로 Proxy되는 리졸버의 입력을 변경시킬 수 있다

```ts
import { Root, Args, Context, Info } from '@nestjs/graphql'
import { Resolve , MiddlewareResolver } from 'nest-graphql-tools'
  // class ... {
  @Shield(and(isAdm, limitRule))
  @PassThrough('Query')
  async getChatMessages(@Resolve() resolve: MiddlewareResolver, @Args() args: any) {
    console.log("Will Proxy Request To Remote Shema using resolve(...)")
    // 이전 예제와 마찬가지로 Proxy 되는 쿼리의 argument , context, info 등을 Transform 할 수 있다
    return resolve({args: ...args, limit: 10 });
  }
```

---

## [GatewayResolver 확장 Class] HasuraShieldRule

- [Hasura](https://hasura.io/)를 원격 스키마로 사용하는 경우 모델단위로 모든 Query, Mutation, Subscription 에 대한 shield rule 을 간편하게 만들어준다.
- @GatewayResolver() 가 적용된 클래스에만 사용 가능하다.
- HasuraShieldRule 추상 클래스를 extend 하여 빌더 함수로 룰을 정의한다
- passthroughRules 메서드를 override 하여 룰을 정의한다

```ts
@GatewayResolver()
class MyHasuraResolver extends HasuraShieldRule {
  override passthroughRules(): HasuraShieldRule {
    return this.for("user")
      .applyAllForQueries(allow, { limit: 50 })
      .applyAllForMutations(deny)
      .for("post")
      .applyAllForQueries(allow)
      .applyAllForMutations(deny)
      .field("room_id", deny);
  }

  @Query()
  ... 중략
}
```

for(model).applyAllForQueries(rule, limitRuleOption) 적용시
- query model
- query model_by_pk
- query model_aggregate
- subscription model
- subscription model_by_pk
- subscription model_aggregate

에 해당 룰들이 적용된다
<br/>

for(model).applyAllForMutations(rule) 적용시
- mutation delete_model
- mutation delete_model_by_pk
- mutation update_model
- mutation update_model_by_pk
- mutation insert_model
- mutation insert_model_by_pk

에 해당 룰들이 적용된다

이외로 update, insert, delete, add(특정 쿼리/뮤테이션 필드 하나 또는 배열) 등이 있으니 인터페이스를 참고한다.

## [GatewayResolver 확장 Class] BaseShieldRule

- 작성 예정

---

## [클래스 데코레이터] GlobalMiddleware

- 모든 리졸버에 적용되는 미들웨어이다
- @GlobalMiddleware() 클래스 데코레이터를 사용한다
- IGqlMiddleware 를 implement하는 클래스를 정의한다

```ts
type GlobalMiddleware = (...operations : ApolloOperations) => ClassDecorator

@GlobalMiddleware('Query', 'Mutation')
export class LoggingGqlMiddleware implements IGqlMiddleware {
  fragment?: string;
  async handleMiddleware(
    @Resolve() resolve: MiddlewareResolver,
    @Info() info: GraphQLResolveInfo
  ): Promise<any> {
    const now = Date.now();
    const resolved = await resolve();
    const timeTaken = Date.now() - now;
    console.log(`Query ${info.fieldName} took ${timeTaken}ms to resolve`);
    return resolved;
  }
}
```

---

## [클래스 데코레이터] InterfaceResolver

- 작성 예정

---

## [클래스 데코레이터] UnionResolver

- 작성 예정
