### [NestJS] Transaction 관련 한번 정리

> 전부 완료되거나, 하나도 완료되지 않아야 하는 특성을 가짐. all or nothing
> 
> 데이터베이스 상태를 변경시키기 위해 수행하는, 일련의 작업으로 함께 처리되어야 하는 작업 단위

1. Typeorm에서 Transaction 처리하는 방법

   1. Transaction 데코레이터 이용

      TypeORM 0.3.x 버전 이후로는 deprecated되었다. 사용을 권장하지 않음

   2. DataSource / EntityManager의 transaction() 메소드 이용

      콜백함수를 인자로 받는다. 이때 첫번째 인자로 트랜잭션 범위 내에서 사용할 수 있는 EntityManager가 주어진다.

      ```jsx
      await this.dataSource.manager.transaction(
        async (transactionalEntityManager) => {
          // 트랜잭션 내에서만 사용할 수 있는 manager
          await transactionalEntityManager.save(User, userData);
          await transactionalEntityManager.save(Profile, profileData);
        }
      );
      ```

   3. QueryRunner 이용

      더 세밀하게 트랜잭션을 제어하고 싶은 경우 사용하는 방식. try-catch-finally와 같이 사용하고, 직접 트랜잭션을 맺고 release 해야 한다.

      ```jsx
      async complexTransaction() {
        const queryRunner = this.dataSource.createQueryRunner();
        await queryRunner.connect();
        await queryRunner.startTransaction();

        try {
          const user = queryRunner.manager.create(User, { name: 'Jane' });
          await queryRunner.manager.save(user);

          // ... 다른 쿼리

          await queryRunner.commitTransaction();
        } catch (err) {
          await queryRunner.rollbackTransaction();
        } finally {
          await queryRunner.release();
        }
      }
      ```

2. 각각의 장단점

| 방식                              | 설명                                                                                                   | 장점                                                                                     | 단점                                                                                             |
| --------------------------------- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **1. `@Transaction` 데코레이터**  | 메서드에 데코레이터를 붙여 트랜잭션 자동 처리                                                          | - 코드가 간결함<br>- 자동으로 트랜잭션 감쌈                                              | - **TypeORM 0.3.x부터 deprecated**<br>- NestJS와 결합 시 유연성 부족                             |
| **2. `DataSource.transaction()`** | 콜백 함수 내에서 트랜잭션을 실행하고 `EntityManager`를 인자로 받음                                     | - 간단한 트랜잭션 처리에 적합<br>- NestJS 서비스 메서드와 쉽게 통합                      | - 복잡한 흐름(예: 중첩 호출)에서 유연성 떨어짐<br>- 트랜잭션 범위가 콜백으로 제한                |
| **3. `QueryRunner` 사용**         | 트랜잭션 수동 제어 (`connect()`, `startTransaction()`, `commitTransaction()`, `rollbackTransaction()`) | - 트랜잭션 전파/중첩 처리 가능<br>- 세밀한 제어 가능<br>- Interceptor 등으로 추상화 가능 | - 코드가 장황하고 복잡<br>- 매번 커넥션 연결/해제 필요<br>- 실수할 여지 많음 (커밋/롤백 누락 등) |

3. queryRunner를 이용한 트랜잭션 구현 - Interceptor로 처리하기

   트랜잭션 처리를 위해 매번 반복적인 try-catch 구문을 사용하는 것은 코드의 중복을 유발하고 유지보수를 어렵게 만든다. 또한 트랜잭션의 시작, 커밋, 롤백을 일관되게 적용하는 데 한계가 있기 때문에 예외처리 누락 등의 위험이 발생할 수 있다.

   이를 개선하기 위해 NestJS의 **Custom Decorator**와 **Interceptor**를 활용하면, 트랜잭션 시작과 종료를 자동화하고, 예외 발생 시 자동으로 롤백되도록 추상화할 수 있다.

   ```jsx
   @Injectable()
   export class TransactionInterceptor implements NestInterceptor {
   constructor(private readonly dataSource: DataSource) {}

   async intercept(
       context: ExecutionContext,
       next: CallHandler,
   ): Promise<Observable<any>> {
       const queryRunner = this.dataSource.createQueryRunner();
       await queryRunner.connect();
       await queryRunner.startTransaction();

       const request = context.switchToHttp().getRequest();
       request.queryRunnerManager = queryRunner.manager;

       return next.handle().pipe(
       catchError(async (error) => {
           await queryRunner.rollbackTransaction();
           await queryRunner.release();

           if (error instanceof HttpException) {
           throw new HttpException(error.getResponse(), error.getStatus());
           }
           throw new InternalServerErrorException();
       }),
       tap(async () => {
           await queryRunner.commitTransaction();
           await queryRunner.release();
       }),
       );
   }
   }
   ```

   그리고 이 인터셉터와 함께 사용할 커스텀 데코레이터도 구현한다.

   ```jsx
   export const TransactionManager = createParamDecorator(
     (data: unknown, ctx: ExecutionContext) => {
       const req = ctx.switchToHttp().getRequest();
       return req.queryRunnerManager;
     }
   );
   ```

   이제 전달받은 request에 있는 queryRunnerManager를 사용하도록 컨트롤러에 다음과 같이 적용해주면 된다 🔍

   ```jsx
   @Controller('')
   export class SomeController {
   constructor(private readonly someService: SomeService) {}

   @Post()
   @UseInterceptors(TransactionInterceptor)
   async createOrder(
       @TransactionManager() transactionManager: EntityManager,
       @Body() body: SomeRequestDto,
   ): Promise<Order> {
       return await this.someService.something(transactionManager, body);
   }
   }
   ```
