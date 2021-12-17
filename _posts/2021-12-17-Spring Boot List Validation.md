---
title: "Spring Boot List Validation"    
layout: single    
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: Spring Boot List Validation
last_modified_at: 2021-12-17
---



Controller에서 요청을 받을 때 다음과 같이 List Object를 파라미터로 받아야 할 경우가 있습니다.

```java
@LoginCheck(userLevel = UserLevel.CUSTOMER)
@PostMapping("/project-apply/{projectPositionId}")
public ResponseEntity<?> modifyProject(@PathVariable("projectPositionId") long projectPositionId,
                                       @RequestBody @Valid List<ProjectApplyRequestDto> projectApplyRequestDto) {
  long userId = SessionUtils.getLoginSessionUserId();

  projectService.projectApply(userId, projectPositionId, projectApplyRequestDto.getList());

  return ResponseEntity.status(HttpStatus.OK).build();
}
```

이때 ProjectApplyRequestDto는 아래와 같이 Validation 검증을 위한 어노테이션이 선언되어있지만 실제로 요청시에는 검증이 되지 않습니다.

```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProjectApplyRequestDto {

    @Min(value = 1, message = "질문 아이디는 필수 입니다.")
    private long applyQuestionId;

    @NotBlank(message = "답변을 입력해주세요.")
    private String answer;
}
```

그 이유는 Object안에 선언된 Object를 검증하기 위해서는 클래스내부에서 검증하고자 하는 Object에 @Valid 어노테이션이 작성 되어있어야 하지만 List의 구현체들에는 그러한 어노테이션이 붙어있지 않기 때문입니다.



이를 해결하는 방법은 생각보다 간단한데 다음처럼 Controller 클래스에 @Validated 어노테이션을 붙여주면 됩니다.

```java
@RestController
@RequiredArgsConstructor
@Validated
public class ProjectController {
  
    @LoginCheck(userLevel = UserLevel.CUSTOMER)
    @PostMapping("/project-apply/{projectPositionId}")
    public ResponseEntity<?> modifyProject(@PathVariable("projectPositionId") long projectPositionId,
                                           @RequestBody @Valid ValidList<ProjectApplyRequestDto> projectApplyRequestDto) {
        long userId = SessionUtils.getLoginSessionUserId();

        projectService.projectApply(userId, projectPositionId, projectApplyRequestDto.getList());

        return ResponseEntity.status(HttpStatus.OK).build();
    }
}

```

@Validated은 스프링에서 제공하는 유효성 검증을 위한 어노테이션 입니다.

보통 검증하고자 하는 대상을 그룹화해서 원하는 그룹만을 검증하고자 할때 많이 사용되지만 위의 예처럼 List 타입의 파라미터에 요청 데이터를 바인딩할 때 유효성 검증을 수행 해주는 역할도 합니다.



하지만 @Validated는 스프링에 종속적인 어노테이션이라는 점이 있습니다.

보통 사용하는 @Valid는 기존 자바 표준기술 이기에 다른 프레임워크나 라이브러리에 종속적이지 않습니다.

그리고 아래와 같은 유효성 검증이 제대로 이루어지는지 테스트를하기위한 코드를 작성했을 때 테스트가 제대로 이루어지지 않습니다.

```java
@Test
@DisplayName("프로젝트 신청 시 질문 아이디 미 입력시 등록 실패")
void projectApplyWithoutApplyQuestionId() {
  List<ProjectApplyRequestDto> projectApplyRequestDto = new ArrayList<>();
  projectApplyRequestDto.add(new ProjectApplyRequestDto(0, "ANSWER_1"));

  Set<ConstraintViolation<List<ProjectApplyRequestDto>>> violations = validator.validate(projectApplyRequestDto);
  ConstraintViolation<List<ProjectApplyRequestDto>> violation = violations.iterator().next();

  assertEquals("질문 아이디는 필수 입니다.", violation.getMessage());
}
```



이를 해결할 수 있는 방법은 List 인터페이스를 확장해 프로퍼티에 @Valid 어노테이션을 정의한 새로운 클래스를 만드는 것 입니다.

```java
@Getter
@Setter
public class ValidList<E> implements List<E> {

    @Valid
    private List<E> list;

    public ValidList() {
        this.list = new ArrayList<>();
    }

    public ValidList(List<E> list) {
        this.list = list;
    }

    @Override
    public int size() {
        return list.size();
    }

    @Override
    public boolean isEmpty() {
        return list.isEmpty();
    }
  
  	.. 나머지 메소드 Override
}
```

ValidList는 프로퍼티에 @Valid 어노테이션이 정의되어있기 때문에 이제 Controller에서 ValidList로 요청 데이터를 바인딩 받으면 유효성 검증이 수행되게 됩니다. 

