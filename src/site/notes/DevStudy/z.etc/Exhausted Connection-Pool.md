---
{"dg-publish":true,"permalink":"/DevStudy/z.etc/Exhausted Connection-Pool/","noteIcon":"","created":"2025-06-05T08:57:16.180+09:00","updated":"2025-06-05T08:57:16.192+09:00"}
---

> 커넥션 풀 마르는 트러블 슈팅

### 문제점 및 해결 과정

> [!WARNING] ❌문제점 1 - Resource 누수 오류 : 커넥션 풀 마르는 현상

Spring batch를 활용한 백업을 하며 S3에 저장된 여러개의 CSV파일을 읽는 작업을 수행 시도하던 중, 
Timeout 오류가 발생했습니다. 에러 로그를 확인해보니 커넥션 풀의 문제가 있음을 발견했습니다.

**✅원인 분석** 
S3객체를 연결하는 문제가 발생하는 코드는 다음과 같았습니다.
```java
private List<Resource> getS3InputStreamResources(List<S3Object> s3Objects) {  
  List<Resource> resources = new ArrayList<>();  
  for (S3Object s3Object : s3Objects) {  
    GetObjectRequest getObjectRequest = GetObjectRequest.builder()  
        .bucket(s3Properties.bucket())  
        .key(s3Object.key())  
        .build();  
	  ResponseInputStream<GetObjectResponse> ri = s3Client.getObject(getObjectRequest); 
    resources.add(new InputStreamResource(ri));  
  }
	return resources;
```
이 코드는 `s3Client.getObject()` 호출 시점에 **즉시 S3와 연결을 열고 InputStream을 생성**합니다. 그리고 이 열린 스트림을 `InputStreamResource`로 감싸 리스트에 담아 `MultiResourceReader`에 전달하는 구조입니다.
문제는 `MultiResourceReader`가 파일을 처리하기도 전에, S3가 열린다는 것입니다. 100개의 s3Object객체라면 반복문에서 100개의 커넥션이 한꺼번에 열리는 구조로,  실행 도중 커넥션 풀이 말라버려 Timeout예외가 발생한 것입니다.
정리하자면, Spring bathc의 구현체가 여러 s3Obejct들을 다룰 때는 순차적으로 읽으며 리소스를 관리하지만 이 구현체에 Resource객체가 도달하기 전에 커넥션이 전부 열려 문제가 발생한 것입니다.
 

**✅ 해결 - 지연 로딩 방식으로 Resouces저장** 
```java
private final ApplicationContext resourceLoader;

		private List<Resource> getS3InputStreamResources(List<S3Object> s3Objects) {  
		  List<Resource> resources = new ArrayList<>();  
		  for (S3Object s3Object : s3Objects) {  
		    String location = "s3://" + s3Properties.bucket() + "/" + s3Object.key();  
		    Resource resource = resourceLoader.getResource(location);
		    resources.add(resource);  
		  }  
		  return resources;  
```
결국 이 문제를 해결하기 위해서는 MultiResourceReader에 도달하기 전에 커넥션이 열리지 않도록 하는 것입니다. 이에 대한 해결 방법으로, 위의 코드와 같은 방법을 택했습니다.
이 방법은 Spring의 `ApplicationContext`를 통해 S3 리소스를 **지연 로딩 방식으로 처리**하는 것입니다.
이 방식으로 변경함으로써 Resource생성 시점에 커넥션이 맺어지지 않고,  `MultiResourceItemReader`가 실제 `getInputStream()`을 호출할 때 S3 **커넥션이 순차적으로 열렸으며, 파일 읽기 종료 후 커넥션이 즉시 반환 처리가**되었습니다.
결과적으로 커넥션 풀 고갈 문제가 말끔히 해결되었고, **성공적으로 수십-수백개의 S3 객체를 처리하는 대용량 배치 작업이 안정적으로 수행**되었습니다.
