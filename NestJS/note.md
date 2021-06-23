---
title: 'NestJS note'
disqus: hackmd
---

NestJS note
===
![downloads](https://img.shields.io/github/downloads/atom/atom/total.svg)
![build](https://img.shields.io/appveyor/ci/:user/:repo.svg)
![chat](https://img.shields.io/discord/:serverId.svg)

## Table of Contents

[TOC]

Config
===
透過nestjs中集成的套件@nestjs/config來處理
```typescript=
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```
透過上述的moduel導入會將.env的配置分配置process.env中。
## module全局使用
依照nestjs設計在各module要使用其他的module必須進行導入，但通常config在各module需要讀取配置時都需使用，但在每個module都import又麻煩這時可以將configModule設置成global就省去每個module導入的步驟。
```typescript=
ConfigModule.forRoot({
isGlobal: true,
})
```

## 自定義配置文件
在複雜得項目，我們可以自定義配置方便自行分組或是加入預設值。建立config/configuration.ts
```typescript=
export default () => ({
  env: process.env.NODE_ENV || 'development',
  name: process.env.APP_NAME || 'PDS-WEB-APP',
  logger: {
    directory: process.env.LOG_DIRECTORY || 'logs/',
    fileName: process.env.LOG_FILENAME,
    level: process.env.LOG_LEVEL || 'info',
  },
  arangoDB: {
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 8529,
    username: process.env.DB_USERNAME || 'pds',
    password: process.env.DB_PASSWORD || 'belstar123',
    dbName: process.env.DB_DATABASE_NAME || 'PDS',
  },
  coreServer: {
    host: process.env.PDS_HOST || 'localhost',
    port: process.env.PDS_PORT || 8080,
  },
});
```
:::info
透過==load==在ConfigModule.forRoot中匯入
:::
```typescript=
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';

@Module({
  imports: [ConfigModule.forRoot({
      load: [configuration],
      isGlobal: true,
  })],
})
export class AppModule {}
```

## ConfigService使用
要從configService取得配置，必須先注入configService。由於我們已經先將ConfigModule設置為Global所以不需要進行import ConfigModule的動作。我們只需在要使用service中透過constructor將configService進行注入。
```typescript=
constructor(private readonly configService: ConfigService) {}
```
在class中可以使用configService.get()取得配置。
```typescript=
this.configService.get('arangoDB.host')
```
[其他參考]([https:// "title](https://docs.nestjs.com/techniques/configuration)")

Winston log
===
Winston log的實現我們將使用nest-winston、winston-daily-rotate-file來配置生產時生成log檔。部分log配置由配置檔讀取，故此範例透過WinstonModule.forRootAsync()異步方法實現。
```typescript=
@Module({
  imports: [
    WinstonModule.forRootAsync({
      useFactory: (configService: ConfigService) => {
        const {
          directory,
          fileName: fName,
          level,
        } = configService.get<Record<string, string>>('logger');
        const fileName =
          fName || `${configService.get('name')}.${configService.get('env')}`;
        const { format, transports } = winston;
        const { combine, timestamp, ms, uncolorize } = format;
        const trans = [];
        trans.push(
          new DailyRotateFile({
            filename: `${fileName}.%DATE%.log`,
            dirname: `${process.cwd()}/${directory}`,
            datePattern: 'YYYY-MM-DD',
            zippedArchive: true,
          }),
        );
        if (configService.get('env') !== 'production') {
          trans.push(
            new transports.Console({
              level: 'debug',
              format: combine(
                timestamp(),
                ms(),
                nestWinstonModuleUtilities.format.nestLike(),
              ),
            }),
          );
        }

        return {
          level,
          format: combine(
            timestamp(),
            ms(),
            nestWinstonModuleUtilities.format.nestLike(),
            uncolorize(),
          ),
          transports: trans,
        };
      },
      inject: [ConfigService],
    }),
    ConfigModule.forRoot({
      load: [configuration],
      isGlobal: true,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
useFactory中回傳的值與winston [createLogger](https://github.com/winstonjs/winston#usage)中的options相同。

接下來我們需要把nestjs的logger替換掉。在main.ts中透過app.get(WINSTON_MODULE_NEST_PROVIDER)取得並使用app.useLogger()進行替換。
```typescript=
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ConfigService } from '@nestjs/config';
import { WINSTON_MODULE_NEST_PROVIDER } from 'nest-winston';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  app.useLogger(app.get(WINSTON_MODULE_NEST_PROVIDER));
  await app.listen(configService.get<string>('APP_PORT'));
}
bootstrap();
```

接著就可如同使用其他service一樣在constructor中進行注入使用。
[參考](https://www.npmjs.com/package/nest-winston)

# GraphQL
Nestjs中使用graphql有兩種方式，一種為code first、schema first。

code first透過typescript class與decorators來生成相應的gql schema。

schema first則是透過GraphQL SDL自動生成相應的typescript class，已進行後續的nestjs撰寫。以下說明皆以code first為主。
## graphql module 配置
建立graphql.module.ts導入GraphQLMoudle並使用期靜態method ==forRoot==進行配置
```typescript=
@Module({
  imports: [
    GraphQLModule.forRoot({
      autoSchemaFile: 'schema.gql',
      uploads: false,
    }),
    AccModule,
    JobModule,
    AuthModule,
  ],
})
export class GraphqlModule {}
```
在配置加入autoSchemaFile指定自動生成.gql檔案位置或是設置成true schema則會即時生成在記憶體中不會產生.gql檔。
## Graphql SDL定義
graphql 中由 object、query、mutation、resolver組成。

### Object Type
code first定義 schema
使用ObjectType decorators對object class做宣告，使用Field來宣告object內的欄位
```typescript=
@ObjectType()
export class Job {
  @Field(() => String)
  jobId: string;

  @Field(() => String)
  sourceName: string;

  @Field(() => Filter)
  filter: Filter;

  @Field(() => String)
  status: string;

  @Field(() => AccContent)
  acc: AccContent;

  @Field(() => String)
  createTime: string;

  @Field(() => Int, { nullable: true })
  resolution?: number;
}
```
:::info
需注意在typescript中只有number可以宣告但是在graphql中有Int和Float，如果不在@Field指定映射類別為Int預設number會映射為Float類別
:::

@Field中options object有以下幾個參數:
* nullable: 用於指定欄位是否為空(未設置預設為皆不可為空) 。true、false、"itemsAndList"、 "items"
* description: 設定欄位描述。
* deprecationReason: 設定欄位棄用原因。

以上class會映射出此 graphql SDL
```graphql=
type Job {
  jobId: String!
  sourceName: String!
  filter: Filter!
  status: String!
  acc: AccContent!
  createTime: String!
  resolution: Int
}
```

當我們的project不大時定義的class數量不多時，使用@Field可以明確定義各欄位內容。但當project複雜起來需定義大量class就會變得冗長且難以維護。 這時我們可以透過啟用GraphQL plugin達到簡化聲明class。
ex. 此方式省略@Field宣告同時也能產生一樣的graphql SDL
```typescript=
@ObjectType()
export class Job {
  jobId: string;
  sourceName: string;
  filter: Filter;
  status: string;
  acc: AccContent;
  createTime: string;

  @Field(() => Int, { nullable: true })
  resolution?: number;
}
```

### 如何使用CLI plugin
在我們的nest-cli.json中啟用pulgin配置
```json=
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql/plugin"]
  }
}
```

[參考](https://docs.nestjs.com/graphql/cli-plugin)

### Query、Mutation Type
宣告完Object type再來要定義使用的query、mutation，這裡的使用方式是直接撰寫resolver再利用decorator進行自動產生SDL
```typescript=
@Resolver(() => Job)
export class JobResolver {
  constructor(
    private readonly jobService: JobService,
    private readonly jobRestService: JobRestService,
    private readonly utils: UtilsService,
    private readonly logger: Logger,
  ) {
    this.logger.setContext(JobResolver.name);
  }

  @Query(() => JobConnection, { name: 'jobList' })
  async findAll(@Args() args: JobListArgs) {
    const { offset, pageSize: limit, invalid } = args;
    const query = invalid !== undefined ? { invalid } : {};
    return this.jobService.findAll(query, { offset, limit });
  }

  @Query(() => Job, { name: 'job' })
  async findOne(@Args('jobId', { type: () => String }) jobId: string) {
    return this.jobService.findOne(jobId);
  }

  @Mutation(() => Job)
  async archiveJob(@Args('jobId', { type: () => String }) jobId: string) {
    const job = await this.jobService
      .update(jobId, { invalid: 1 })
      .then((result) => result[0])
      .then((result) => result.new);
    await this.utils.deleteTmpFile(
      `.${job.tmpName}`,
      `./tmp/imposition/${jobId}.pdf`,
    );
    return job;
  }

  @ResolveField()
  jobId(@Parent() job: Job) {
    return job.jobId || job['_key'];
  }
}
```
透過@Resolver對class宣告為Job的解析器，利用@Query、@Mutation對class method進行query及mutation的宣告，以上class會產生出對應的SDL
```graphql=
type Query {
  jobList(
    """The number of results ti show. Must be >= 1. Default = 20"""
    pageSize: Int = 20
    offset: Int = 0
    invalid: Invalid
  ): JobConnection!
  job(jobId: String!): Job!
}

type Mutation {
  archiveJob(jobId: String!): Job!
}
```

:::info
Query、Mutation type中生成名稱預設為class method名稱如果需要調整請在@Query、@Mutation中的options宣告name 
ex. @Query(() => Job, { name: 'job' })
:::

query、mutation中method使用的args透過@Args()進行宣告[詳細請參考](#Input、-Args-Type)

[resolvers參考](https://docs.nestjs.com/graphql/resolvers)
[mutations參考](https://docs.nestjs.com/graphql/mutations)

完成resolver後將寫入module的providers中，及完成一個module建立。
```typescript=
@Module({
  providers: [
    JobResolver,
    JobService,
    JobRestService,
    Logger,
    {
      provide: 'COLLECTION_NAME',
      useValue: 'Job',
    },
    CollectionService,
    UtilsService,
  ],
})
export class JobModule {}
```
此時GraphQLModuel還找不到你定義的各項graphql SDL因為尚未把此module註冊進去，所以需把完成的module導入。
```typescript=
@Module({
  imports: [
    GraphQLModule.forRoot({
      autoSchemaFile: 'schema.gql',
    }),
    JobModule,
  ],
})
export class GraphqlModule {}
```

### Input、 Args Type
在resolver中可以透過@Args()對method args進行宣告透過第一個參數宣告在SDL中的名稱。
```typescript=
@Query(() => JobConnection, { name: 'jobList' })
async findAll(
    @Args('pageSize', { type: () => Int, nullable: true , defaultValue: 20}) pageSize: number,
    @Args('offset', { type: () => Int, nullable: true, defaultValue: 0}) offset: number,
    @Args('invalid', { type: () => Invalid, nullable: true  }) invalid: Invalid) {
  const query = invalid !== undefined ? { invalid } : {};
  return this.jobService.findAll(query, { offset, limit });
}
```
以上會映射成
```graphql=
type Query {
    jobList(
    """The number of results ti show. Must be >= 1. Default = 20"""
    pageSize: Int = 20
    offset: Int = 0
    invalid: Invalid
  ): JobConnection!
}
```
由於每個參數都需要@Args()宣告，當參數數量變多或是是一個Object，我們可以另外抽出來定義
```typescript=
@ArgsType()
export class JobListArgs {
  @Field(() => Int, {
    description: 'The number of results ti show. Must be >= 1. Default = 20',
    nullable: true,
    defaultValue: 20,
  })
  pageSize: number;

  @Field(() => Int, { nullable: true, defaultValue: 0 })
  offset: number;
  invalid?: Invalid;
}
```

:::info
Mutation所使用的input與Query使用的args除了對class所宣告到@InputType()與 @ArgsType()其他欄位宣告均與Objecy Type相同
:::

我們可以將resolver中method修改成:
```typescript=
@Query(() => JobConnection, { name: 'jobList' })
async findAll(@Args() args: JobListArgs) {
  const { offset, pageSize: limit, invalid } = args;
  const query = invalid !== undefined ? { invalid } : {};
  return this.jobService.findAll(query, { offset, limit });
}
```
同樣會映射出相同的SDL，並且我們可以透過抽離args、input使用typescript對其進行後續維護或是相關欄位驗證。

## Enum 使用
先進行enum宣告，再將宣告enum使用registerEnumType進行註冊即可在Object class或是其他地方使用
```typescript=
export enum Invalid {
  enabled = 1,
  disabled = 0,
}

registerEnumType(Invalid, {
  name: 'Invalid',
});
```
[參考](https://docs.nestjs.com/graphql/unions-and-enums#enums)

## Datasource 使用
### Rest Datasource

```typescript=
@Injectable()
export class BaseRestService extends RESTDataSource {
  constructor(
    private configService: ConfigService,
    protected readonly logger: Logger,
    @Inject(CONTEXT) context,
  ) {
    super();
    const { host, port } = this.configService.get('coreServer');
    this.baseURL = `http://${host}:${port}/api/`;
    this.logger.setContext(BaseRestService.name);
    this.context = context;
    this.initialize({ context, cache: new InMemoryLRUCache() });
  }

  protected willSendRequest(request: RequestOptions) {
    if (request.method === 'GET') {
      const cacheKey = this.resolveURL(request).toString();
      if (this.memoizedResults.has(cacheKey))
        this.memoizedResults.delete(cacheKey);
    }
    if (this.context.user) {
      request.headers.set('Authorization', `Bearer ${this.context.user.token}`);
    }
    if (
      request.body &&
      typeof request.body === 'object' &&
      !(request.body instanceof FormData)
    ) {
      request.body = { ...request.body };
    }
  }

  async didReceiveResponse(response) {
    if (response.ok) {
      return this.parseBody(response).then(
        (result: Record<string, any>) => result.data,
      );
    } else {
      const error = await this.errorFromResponse(response);
      throw Boom.boomify(error, {
        message: error.extensions.response.body.error.message,
        statusCode: error.extensions.response.status,
      });
    }
  }
}
```

:::info
apollo server 中用到的context可以使用@Inject(CONTEXT)注入取得， CONTEXT由@nestjs/graphqkl中取得。
:::
:::warning
使用apollo-datasource-rest 套件需注意，透過DI注入方式建立實例，不是直接把dataSource透過options傳遞給Apollo server instance，會發生datasource沒有initialize在發送http請求時會出現 
==TypeError: Cannot read property 'fetch' of undefined== 這邊處理方式是在建構時呼叫initialize進行初始化動作。
:::

### data
###### tags: `Templates` `Documentation`


