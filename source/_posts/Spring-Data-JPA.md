---
title: 【Spring Data JPA】 Solved bad performance about using JpaRepository flush method
---
There are some slow behavior batch features in my project(SpringBoot + Hibernate + Spring-Data-JPA + MySQL).
I'm trying to INSERT 10000 records in the loop into the table by usering flush() method on each 30 records. 
It spent one hour for inserting those datas. And it spent 4 minutes after optimizing.
Here is my code before optimizing.

###### BatchRepository.java

``` bash
@Repository
public interface BatchRepository extends JpaRepository<XXX, String> {
    // query method 
}
```


###### MstEntity.java

``` bash
@Data
@Entity
public class MstEntity implements Serializable {
    // property
}
```


###### UploadService.java

``` bash
private final JpaRepository repository;
private final BatchRepository batchRepository;

@Transactional
public UploadResultResponse save(final MultipartFile req) throws XXXException {

	long registCnt = 0; 

	try (final CsvReader csvReader = new CsvReader(XXX);) {        

	    long nextDbFlushCnt = this.BATCH_SIZE;

	    while (true) {

		final MstEntity entity = this.readCsvAndValid(csvReader, param);

		if (entity == null) {
		    break;
		}

		final Long csvRow = Long.valueOf(csvReader.getReadCnt());

		if (condition1) {

		    final XXX obj1 = this.businessLogicMethod1(entity, csvRow);

		    if (condition2) {

			this.batchRepository.save(obj1);
			registCnt++;
			this.batchRepository.flush();
		    }


		    final List<XXX> obj2 = this.businessLogicMethod2(entity, csvRow);
		    final List<XXX> insertTasks = repository.saveAll(obj2);

		    registCnt += insertTasks.size();
		} else {

		    // business logic
		    XXXXXXXX;

		    repository.save(entity);

		    registCnt++;
		}

		if (nextDbFlushCnt <= registCnt) {

		    repository.flush();

		    nextDbFlushCnt = registCnt + this.BATCH_SIZE;
		    log.trace("batch inserted。inserted until now={}、insert next time={}、insert circle={}",
			      Long.valueOf(registCnt),
			      Long.valueOf(nextDbFlushCnt),
			      Long.valueOf(this.BATCH_SIZE));
		}
	    }
	    repository.flush();
	} catch (final IOException e) {
	    log.error("CSV read error", e);
	    throw new XXXException(e, XXXX);
	}


	// business logic
    XXXXXXXX;
}
```


###### application.yml

``` bash
...
jpa
    properties
        hibernate.order_inserts: true
        hibernate.order_updates: true
        hibernate.jdbc.batch_size: 30
...
```
## ★Solution
#### Step 1:
add 「hibernate.generate_statistics: true」setting in your application.yml file.

``` bash
...
jpa
    properties
        hibernate.order_inserts: true
        hibernate.order_updates: true
        hibernate.jdbc.batch_size: 30
        hibernate.generate_statistics: true // add this
...
```

#### Step 2:
Check console logs before optimizing.
JDBC Total execute Time: 98min

``` bash
9498503900 nanoseconds spent preparing 150158 JDBC statements;
58258549600 nanoseconds spent executing 150167 JDBC statements;
5623197400 nanoseconds spent executing 10241 JDBC batches;
445898982300 nanoseconds spent executing 10936 flushes (flushing a total of 52600451 entities and 374430 collections);
5325683486200 nanoseconds spent executing 130386 partial-flushes (flushing a total of 662516359 entities and 662516359 collections)
```

#### Step 3:
Add EntityManager.clear() method after each repository.flush() called.

###### UploadService.java

``` bash

@PersistenceContext
private EntityManager entityManager; // add EntityManager
private final JpaRepository repository;
private final BatchRepository batchRepository;

@Transactional
public UploadResultResponse save(final MultipartFile req) throws XXXException {
    
    long registCnt = 0; 

    try (final CsvReader csvReader = new CsvReader(XXX);) {        
                                                    
        long nextDbFlushCnt = this.BATCH_SIZE;

        while (true) {
        
            final MstXXX entity = this.readCsvAndValid(csvReader, param);
            
            if (entity == null) {
                break;
            }
            
            final Long csvRow = Long.valueOf(csvReader.getReadCnt());

            if (condition1) {
                
                final XXX obj1 = this.businessLogicMethod1(entity, csvRow);
                
                if (condition2) {

                    this.batchRepository.save(obj1);
                    registCnt++;
                    this.batchRepository.flush();
                    this.entityManager.clear();  // add clear method
                }
                
                
                final List<XXX> obj2 = this.businessLogicMethod2(entity, csvRow);
                final List<XXX> insertTasks = repository.saveAll(obj2);
                
                registCnt += insertTasks.size();
            } else {

                // business logic
                XXXXXXXX;

                repository.save(entity);

                registCnt++;
            }

            if (nextDbFlushCnt <= registCnt) {
            
                repository.flush();
                
                this.entityManager.clear();  // add clear method
                
                nextDbFlushCnt = registCnt + this.BATCH_SIZE;
                log.trace("batch inserted。inserted until now={}、insert next time={}、insert circle={}",
                            Long.valueOf(registCnt),
                            Long.valueOf(nextDbFlushCnt),
                            Long.valueOf(this.BATCH_SIZE));
            }
        }
        repository.flush();
        this.entityManager.clear();  // add clear method
    } catch (final IOException e) {
        log.error("CSV read error", e);
        throw new XXXException(e, XXXX);
    }

    // business logic
    XXXXXXXX;
}
```

#### Step 4
check the console log again. you will find something different.
JDBC Total execute Time: 2.3min

``` bash
6957340700 nanoseconds spent preparing 162204 JDBC statements;
41894343700 nanoseconds spent executing 150186 JDBC statements;
10699438700 nanoseconds spent executing 12351 JDBC batches;
33201743100 nanoseconds spent executing 12355 flushes (flushing a total of 646277 entities and 360396 collections)
47066793200 nanoseconds spent executing 130167 partial-flushes (flushing a total of 7690767 entities and 7690767 collections)
```


■ By the way, here is some userful articles maybe helpful for you
https://terasolunaorg.github.io/guideline/public_review/ArchitectureInDetail/DataAccessJpa.html#spring-data-jpa

■ Offical Site
https://github.com/spring-projects/spring-data-jpa
