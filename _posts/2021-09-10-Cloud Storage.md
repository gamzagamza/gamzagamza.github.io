---
title: "서버가 여러개일때 파일은 어디에 저장할까 (feat. Cloud Storage)"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: GCP Cloud Storage를 사용한 파일 관리
last_modified_at: 2021-09-10     
---


## 다중 서버 환경에서의 파일 관리

현재 Goods Shop의 서버 구성은 다음과 같이 2개의 Spring boot 서버로 이루어져 있습니다.

![1](/assets/image/cloud_storage/1.png)

만약 사용자가 파일이나 이미지등을 업로드했을때 서버의 컴퓨터에 저장하게되면 해당 파일을 다운로드 받으려할때 무조건 파일이 있는 서버로 요청을 보내야 정상적으로 파일을 다운로드 받을 수 있을것입니다.

하지만 현재 로드밸런싱 설정이 Round robin으로 설정되어있기 때문에 무조건 파일이 있는 서버로 요청을 보낸다고 장담할 수 없습니다.

이 문제는 파일이나 이미지등을 업로드할때 하나의 서버에 업로드한 후 가져올때도 하나의 서버에서만 가져오도록 하면 해결됩니다.

요즘에는 클라우드 서비스를 제공하는 업체에서 이러한 파일 서버를 제공해주는데 AWS에서는 s3 GCP에서는 Cloud Storage라는 이름으로 제공되어지고 있습니다.

현재 Google Cloud Platform을 이용중이기 때문에 Cloud Storage를 사용해서 문제를 해결해보겠습니다.



## Cloud Storage

cloud storage란 Object Repository(객체 저장소)를 뜻하며, 데이터의 양과 관계없이 언제 어디서나 데이터를 저장하고 가져올 수 있는 서비스를 말합니다.

- Project : 프로젝트는 하나의 애플리케이션과 관계가 있으며, 각 프로젝트는 고유한 cloud storage api과 resource를 가진다.
- Bucket : 각 프로젝트는 여러개의 Bucket를 가질 수 있으며, Bucket는 데이터를 저장하는 공간이다.
- Object : Bucket에 저장되는 데이터를 의미한다.



### Spring Boot + GCP 설정

GCP Console에 접속해 Cloud Storage 화면으로 이동한 후 Bucket을 생성해보겠습니다. (프로젝트는 이미 생성되어있다고 가정하겠습니다.)

![1](/assets/image/cloud_storage/2.png)

버킷이름은 해당 버킷을 사용하는 프로젝트이름_bucket 으로 설정하였고 데이터 저장위치는 Region에 서울로 하였습니다.

나머지 설정은 Default 설정으로 건드리지 않고 bucket을 생성하겠습니다.

![3](/assets/image/cloud_storage/3.png)



bucket 생성이 완료되었으면 이제 해당 storage를 사용하기위한 서비스 계정을 생성해야 합니다.

IAM 및 관리자 > 서비스 계정으로 이동한 후 서비스 계정을 생성하면되는데 이때 권한설정에 storage관련 권한을 추가해주어야 합니다.

계정 생성이 완료되었으면 목록에서 해당 계정을 클릭 후 상세화면에서 키 탭으로 이동한 후 키를 생성하여 로컬에 해당 키를 다운로드 받습니다.

그리고 다운로드 받은 키를 Spring Boot 프로젝트 resources 폴더로 이동시켜줍니다.

![4](/assets/image/cloud_storage/4.png)

키파일을 이동시킨 후 properties 설정을 통해 해당 경로를 잡아주어야 하는데 다음과 같이 추가해주면 됩니다.

![5](/assets/image/cloud_storage/5.png)

마지막 설정으로 pom.xml에 관련 의존성을 추가해주면 됩니다.

![6](/assets/image/cloud_storage/6.png)

이제 스프링을 실행해보고 아무런 오류가 발생하지 않으면 정상적으로 설정이 완료된 것 입니다.



### Download & Upload Source Code(Java)

다운로드와 업로드를 위한 유틸 클래스를 만들어 사용했습니다.

```java
@Component
public class GcpStorageUtils {

    @Autowired
    private Storage storage;

    public void uploadGcs(MultipartFile multiFile, FileDetailDTO fileDetailDTO) throws IOException {
        BlobInfo blobInfo = storage.create(
                BlobInfo.newBuilder("xoov_bucket", fileDetailDTO.getPath()).build(),
                multiFile.getBytes(),
                Storage.BlobTargetOption.predefinedAcl(Storage.PredefinedAcl.PUBLIC_READ)
        );
    }

    public InputStream downloadGcs(FileDetailDTO fileDetailDTO) {
        Blob blob = storage.get("xoov_bucket", fileDetailDTO.getPath());
        ReadChannel readChannel = blob.reader();

        return Channels.newInputStream(readChannel);
    }
}
```

업로드시에는 파일의 바이트 배열을 가져오기위해 MultiPartFile객체와 저장하려는 파일의 정보를 담은 FileDetailDTO객체를 매개변수로 받아 storage에 저장합니다.



다운로드시에는 버킷명과 해당 버킷에서 가져올 파일의 경로와 파일명을통해 파일의 Blob 객체를 가져온뒤 InputStream객체를 얻어와 리턴하도록 하였습니다.

리턴된 InputStream을 포함시켜 컨트롤러에서 응답을 전송하면 다운로드가 정상적으로 진행됩니다.

```java
@RequestMapping("/file")
@Controller
public class FileController {

    @Autowired
    private FileService fileService;

    @Autowired
    private GcpStorageUtils gcpStorageUtils;

    @GetMapping("/download/{fileId}/{fileSn}")
    public ResponseEntity<Resource> download(@PathVariable("fileId")Long fileId,
                                             @PathVariable("fileSn")Integer fileSn) throws FileNotFoundException {
        FileDetailDTO fileDetailDTO = fileService.getDetailFile(fileId, fileSn);

        InputStreamResource resource = new InputStreamResource(gcpStorageUtils.downloadGcs(fileDetailDTO));

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + fileDetailDTO.getOriginalName() + "\"")
                .contentType(MediaType.parseMediaType("application/octet-stream"))
                .body(resource);
    }
}
```

