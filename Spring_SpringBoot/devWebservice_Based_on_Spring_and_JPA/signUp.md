# 회원가입과 로그인

## 계정 관리 기능 미리보기
- 회원 가입
- 이메일 인증
- 로그인
- 로그아웃
- 프로필 추가 정보 입력
- 프로필 이미지 등록
- 알림 설정
- 패스워드 수정
- 패스워드를 잊어버린 경우
- 관심 주제(태그) 등록
- 주요 활동 지역 등록
<br>

## 계정 도메인
- domain 패키지에 Account 도메인 생성
    * `@EqualsAndHashCode(of = "id")`
        - 연관관계가 복잡해지면 `equals()`, `hashcode()`에서 서로 다른 연관관계를 계속해서 순환 참조하느라 무한 루프가 발생하고 결국에는 StackOverFlow가 발생할 수 있다.
        - 따라서 id만 사용 
    * email과 nickname은 중복 되면 안된다.(unique)
    * `@Lob @Basic(fetch = FetchType.EAGER)`
        - 프로필 이미지는 VARCHAR(255)보다 커질 가능성이 있고
        - 유저를 로딩할 때 종종 같이 쓰이므로 바로바로 가져올 수 있도록 즉시 로딩 사용
```java
@Entity
@Getter @Setter @EqualsAndHashCode(of = "id")
@Builder @AllArgsConstructor @NoArgsConstructor
public class Account {

    @Id @GeneratedValue
    private Long id;

    @Column(unique = true)
    private String email;

    @Column(unique = true)
    private String nickname;

    private String password;

    private boolean emailVerified; // 이메일 인증 절차 유무를 판단할 flag

    private String emailCheckToken; // 이메일을 검증할 때 사용할 토큰 값

    private LocalDateTime joinedAt; // 인증을 거친 사용자들의 가입 날짜

    private String bio; // 간단한 자기소개

    private String url; // 웹사이트 url

    private String occupation; // 직업

    private String location; // 거주 지역

    @Lob @Basic(fetch = FetchType.EAGER)
    private String profileImage; // 프로필 이미지

    private boolean studyCreatedByEmail; // 스터디가 생성되면 이메일로 받을 것인지

    private boolean studyCreatedByWeb; // 스터디가 생성되면 웹으로 받을 것인지

    private boolean studyEnrollmentResultByEmail; // 스터디 가입 신청 결과를 이메일로 받을 것인지

    private boolean studyEnrollmentResultByWeb; // 스터디 가입 신청 결과를 웹으로 받을 것인지

    private boolean studyUpdatedByEmail; // 스터디 변경 정보를 이메일로 받을 것인지

    private boolean studyUpdatedByWeb; // 스터디 변경 정보를 웹으로 받을 것인지

}
```
<br>

## 회원가입 컨트롤러

### 목표
- GET “/sign-up” 요청을 받아서 account/sign-up.html 페이지 보여준다.
- 회원 가입 폼에서 입력 받을 수 있는 정보를 “닉네임", “이메일", “패스워드" 폼 객체로 제공한다
<br>

### 구현
- account 패키지 생성 후 AccountController 생성
```java
@Controller
public class AccountController {

    @GetMapping("/sign-up")
    public String signUpForm(Model model) {
        return "account/sign-up";
    }
}
```
- 실행하면 다음과 같이 login 화면이 나온다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_1.jpg"></p>

- 매번 localhost:8080에 접속할 때마다 login으로 redirect되기 때문에 Spring Security 설정 추가
    * config 패키지를 만들고 SecurityConfig 추가
    * `@EnableWebSecurity`
        - Spring Security 설정을 직접 한다.
    * 설정한 특정 요청은 Security 인증하지 않도록 설정하고 나머지는 로그인 해야만 하도록 설정
    * profile은 GET만 허용
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .mvcMatchers("/", "/login", "/sign-up", "/check-email", "check-email-token",
                        "/email-login", "/check-email-login", "/login-link").permitAll()
                .mvcMatchers(HttpMethod.GET, "profile/*").permitAll()
                .anyRequest().authenticated();
    }
}
```
- 실행하면 로그인 화면(/login)으로 가지 않는 것을 확인할 수 있다.
- 테스트 코드 작성
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @DisplayName("회원 가입 화면이 보이는지 테스트")
    @Test
    void signUpForm() throws Exception {
        mockMvc.perform(get("/sign-up"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"));
    }

}
```
<br>

## 회원가입 뷰

### 목표
- Bootstrap
    * 네이게이션 바 만들기
    * 폼 만들기
- Thymeleaf
    * SignUpForm 타입 객체를 폼 객체로 설정하기
- 웹(HTML, CSS, JavaScript)
    * 제약 검증 기능 사용하기
        - 닉네임 (3~20자, 필수 입력)
        - 이메일 (이메일 형식, 필수 입력)
        - 패스워드 (8~50자, 필수 입력)
<br>

### 구현
- 회원가입 뷰 작성
    * 참고로 뷰에서 사용하고 있는 icon 이미지 파일을 넣어줘야 한다.(templates/static/images)
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>StudyOlle</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.5.3/dist/css/bootstrap.min.css" integrity="sha384-TX8t27EcRE3e/ihU7zmQxVncDAy5uIKz4rEkgIXeMed4M0jlfIDPvg6uqKI2xXr2" crossorigin="anonymous">
    <style>
        .container {
            max-width: 100%;
        }
    </style>
</head>

<body class="bg-light">
    <nav class="navbar navbar-expand-sm navbar-dark bg-dark">
        <a class="navbar-brand" href="/" th:href="@{/}">
            <img src="/images/logo_sm.png" width="30" height="30">
        </a>
        <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav mr-auto">
                <li class="nav-item">
                    <form th:action="@{/search/study}" class="form-inline" method="get">
                        <input class="form-control mr-sm-2" name="keyword" type="search" placeholder="스터디 찾기" aria-label="Search" />
                    </form>
                </li>
            </ul>

            <ul class="navbar-nav justify-content-end">
                <li class="nav-item">
                    <a class="nav-link" href="#" th:href="@{/login}">로그인</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="#" th:href="@{/signup}">가입</a>
                </li>
            </ul>
        </div>
    </nav>

    <div class="container">
        <div class="py-5 text-center">
            <h2>계정 만들기</h2>
        </div>
        <div class="row justify-content-center">
            <form class="needs-validation col-sm-6" action="#"
                  th:action="@{/signup}" th:object="${signUpForm}" method="post" novalidate>
                <div class="form-group">
                    <label for="nickname">닉네임</label>
                    <input id="nickname" type="text" th:field="*{nickname}" class="form-control"
                           placeholder="닉네임" aria-describedby="nicknameHelp" required minlength="3" maxlength="20">
                    <small id="nicknameHelp" class="form-text text-muted">
                        공백없이 문자와 숫자로만 3자 이상 20자 이내로 입력하세요. 가입후에 변경할 수 있습니다.
                    </small>
                    <small class="invalid-feedback">닉네임 형식이 올바르지 않습니다.</small>
                    <small class="form-text text-danger" th:if="${#fields.hasErrors('nickname')}" th:errors="*{nickname}">Nickname Error</small>
                </div>

                <div class="form-group">
                    <label for="email">이메일</label>
                    <input id="email" type="email" th:field="*{email}" class="form-control"
                           placeholder="example@email.com" aria-describedby="emailHelp" required>
                    <small id="emailHelp" class="form-text text-muted">
                        스터디올래는 사용자의 이메일을 공개하지 않습니다.
                    </small>
                    <small class="invalid-feedback">이메일 형식이 올바르지 않습니다.</small>
                    <small class="form-text text-danger" th:if="${#fields.hasErrors('email')}" th:errors="*{email}">Email Error</small>
                </div>

                <div class="form-group">
                    <label for="password">패스워드</label>
                    <input id="password" type="password" th:field="*{password}" class="form-control"
                           aria-describedby="passwordHelp" required minlength="8" maxlength="50">
                    <small id="passwordHelp" class="form-text text-muted">
                        8자 이상 50자 이내로 입력하세요. 영문자, 숫자, 특수기호를 사용할 수 있으며 공백은 사용할 수 없습니다.
                    </small>
                    <small class="invalid-feedback">패스워드 형식이 올바르지 않습니다.</small>
                    <small class="form-text text-danger" th:if="${#fields.hasErrors('password')}" th:errors="*{password}">Password Error</small>
                </div>

                <div class="form-group">
                    <button class="btn btn-primary btn-block" type="submit"
                            aria-describedby="submitHelp">가입하기</button>
                    <small id="submitHelp" class="form-text text-muted">
                        <a href="#">약관</a>에 동의하시면 가입하기 버튼을 클릭하세요.
                    </small>
                </div>
            </form>
        </div>

        <footer th:fragment="footer">
            <div class="row justify-content-center">
                <img class="mb-2" src="/images/logo_long_kr.png" alt="" width="100">
                <small class="d-block mb-3 text-muted">&copy; 2020</small>
            </div>
        </footer>
    </div>
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js" integrity="sha384-DfXdz2htPH0lsSSs5nCTpuj/zy4C+OGpamoFVy38MVBnE+IbbVYUew+OrCXaRkfj" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.1/dist/umd/popper.min.js" integrity="sha384-9/reFTGAW83EW2RDu2S0VKaIzap3H66lZH81PoYlFhbGU+6BZp6G7niu735Sk7lN" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.5.3/dist/js/bootstrap.min.js" integrity="sha384-w1Q4orYjBQndcko6MimVbzY0tgp4pWB4lZ7lr30WKz0vr/aWKhXdBNmNb5D92v7s" crossorigin="anonymous"></script>
    <script type="application/javascript">
        (function () {
            'use strict';

            window.addEventListener('load', function () {
                // Fetch all the forms we want to apply custom Bootstrap validation styles to
                var forms = document.getElementsByClassName('needs-validation');

                // Loop over them and prevent submission
                Array.prototype.filter.call(forms, function (form) {
                    form.addEventListener('submit', function (event) {
                        if (form.checkValidity() === false) {
                            event.preventDefault();
                            event.stopPropagation();
                        }
                        form.classList.add('was-validated')
                    }, false)
                })
            }, false)
        }())
    </script>
</body>
</html>
```
- 회원가입 폼 추가
```java
@Data
public class SignUpForm {

    private String nickname;
    private String email;
    private String password;

}
```
- 컨트롤러에 model 추가
```java
@Controller
public class AccountController {

    @GetMapping("/sign-up")
    public String signUpForm(Model model) {
        // model.addAttribute("signUpForm", new SignUpForm());
        // 생략하면 클래스 이름의 CamelCase를 사용한다.
        model.addAttribute(new SignUpForm());
        return "account/sign-up";
    }
}
```
- 실행하면 이미지 파일들이 Spring Security에 걸려서 허용되지 않아 403 에러가 나온다.
    * 따라서 static resource들은 인증하지 않고 제공하기 위해 설정한다.
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    ...

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring()
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }
}
```
- 테스트 코드 작성
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @DisplayName("회원 가입 화면이 보이는지 테스트")
    @Test
    void signUpForm() throws Exception {
        mockMvc.perform(get("/sign-up"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(model().attributeExists("signUpForm"));

    }
}
```
- 실행하면 다음과 같이 뷰가 잘 만들어진 것을 볼 수 있다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_2.jpg"></p>

<br>

## 회원가입 폼 서브밋 검증

### 목표
- JSR 303 애노테이션 검증
    * 값의 길이, 필수값
- 커스텀 검증
    * 중복 이메일, 닉네임 여부 확인
- 폼 에러 있을 시, 폼 다시 보여주기

### 구현
- **JSR 303 애노테이션 검증하기**
- 폼 서브밋 요청에 대한 Post맵핑 추가
```java
@Controller
public class AccountController {

    ...

    @PostMapping("/sign-up")
    public String signUpSubmit(@Valid SignUpForm signUpForm, Errors errors) {
        if(errors.hasErrors()) {
            return "account/sign-up";
        }
        
        // TODO 회원 가입 처리
        return "redirect:/";

    }
}
```
- 폼 객체에 Validation 애노테이션 추가
```java
@Data
public class SignUpForm {

    @NotBlank
    @Length(min = 3, max = 20)
    @Pattern(regexp = "^[ㄱ-ㅎ가-힣a-z0-9_-]{3,20}$")
    private String nickname;

    @Email
    @NotBlank
    private String email;

    @NotBlank
    @Length(min = 8, max = 50)
    private String password;

}
```
- 실행하면 잘 되지만 다음과 같이 패턴에 어긋나는 값을 넣을 경우 경고 메세지가 나온다.
    * 프론트에서 검증 코드를 넣었지만 뚫린다.
        - 의미가 없진 않다.
        - 서버에 오기 전에 걸러주면 리소스를 절약할 수 있고
        - 사용자에게 빠르게 fail-fast해줄 수 있다.
    * **프론트에서도 검증하고 백엔드에서도 검증을 해야하는 이유다.**
- **커스텀 검증하기**
- SignUpFormValidator 생성
    * `@Component`
        - AccountRepository을 주입받아 사용하려면 bean으로 등록되야 한다.
```java
@Component // AccountRepository을 주입받아 사용하려면 bean으로 등록되야 한다.
@RequiredArgsConstructor
public class SignUpFormValidator implements Validator {

    private final AccountRepository accountRepository;

    @Override
    public boolean supports(Class<?> aClass) {
        return aClass.isAssignableFrom(SignUpForm.class); // 검증할 인스턴스의 타입
    }

    @Override
    public void validate(Object object, Errors errors) {
        SignUpForm signUpForm = (SignUpForm)object;

        if(accountRepository.existsByEmail(signUpForm.getEmail())) {
            errors.rejectValue("email", "invalid.email", new Object[]{signUpForm.getEmail()}, "이미 사용중인 이메일입니다.");
        }

        if(accountRepository.existsByNickname(signUpForm.getNickname())) {
            errors.rejectValue("nickname", "invalid.nickname", new Object[]{signUpForm.getNickname()}, "이미 사용중인 닉네임입니다.");
        }
    }
}
```
- AccountRepository 생성
```java
@Transactional(readOnly = true) // 읽기전용으로 만들어 성능에 조금이라도 이점을 가져온다. 
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    boolean existsByEmail(String email);
    boolean existsByNickname(String nickname);
}
```
- 컨트롤러에 커스텀 검증 사용
    * `@InitBinder("signUpForm")`
        - signUpForm이라는 데이터를 받을 때 바인더를 설정한다.
        - WebDataBinder라는 것을 파라미터로 받고 Validator를 추가할 수 있다.
        - 추가하면 signUpForm을 받을 때 받은 파라미터의 camelCase에 맞춰 validator도 사용이 된다.
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    private final SignUpFormValidator signUpFormValidator;

    @InitBinder("signUpForm")
    public void initBinder(WebDataBinder webDataBinder) {
        webDataBinder.addValidators(signUpFormValidator);
    }

    ...
}
```
<br>

## 회원 가입 폼 서브밋 처리

### 목표
- 회원 가입 처리
    * 회원 정보 저장
    * 인증 이메일 발송
    * 처리 후 첫 페이지로 리다이렉트([Post-Redirect-Get 패턴](https://en.wikipedia.org/wiki/Post/Redirect/Get))

### 구현
- 이메일은 MailSenderAutoConfiguration에 이미 정의 되어있고 MailSender를 주입받아 사용할 수 있지만  아직은 가짜 객체를 만들어서 콘솔에 출력하도록 한다.
    * 실제 객체로는 나중에 구현
```java
@Profile("local")
@Component
@Slf4j
public class ConsoleMailSender implements JavaMailSender {
    @Override
    public MimeMessage createMimeMessage() {
        return null;
    }

    @Override
    public MimeMessage createMimeMessage(InputStream inputStream) throws MailException {
        return null;
    }

    @Override
    public void send(MimeMessage mimeMessage) throws MailException {

    }

    @Override
    public void send(MimeMessage... mimeMessages) throws MailException {

    }

    @Override
    public void send(MimeMessagePreparator mimeMessagePreparator) throws MailException {

    }

    @Override
    public void send(MimeMessagePreparator... mimeMessagePreparators) throws MailException {

    }

    @Override
    public void send(SimpleMailMessage simpleMailMessage) throws MailException {
        log.info(simpleMailMessage.getText());
    }

    @Override
    public void send(SimpleMailMessage... simpleMailMessages) throws MailException {

    }
}
```
- Profile을 local로 설정
    * application.properties
```properties
spring.profiles.active=local
```
- 회원 가입 처리
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    private final SignUpFormValidator signUpFormValidator;
    private final AccountRepository accountRepository;
    private final JavaMailSender javaMailSender;

    ...

    @PostMapping("/sign-up")
    public String signUpSubmit(@Valid SignUpForm signUpForm, Errors errors) {
        
        ...

        // 회원 가입 처리
        Account account = Account.builder()
                .email(signUpForm.getEmail())
                .nickname(signUpForm.getNickname())
                .password(signUpForm.getPassword()) // TODO encoding 해야한다.
                .studyCreatedByWeb(true)
                .studyEnrollmentResultByWeb(true)
                .studyUpdatedByWeb(true)
                .build();

        Account newAccount = accountRepository.save(account);

        newAccount.generateEmailCheckToken();
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setTo(newAccount.getEmail());
        mailMessage.setSubject("스터디올래, 회원 가입 인증"); // 제목
        mailMessage.setText("/check-email-token?token=" + newAccount.getEmailCheckToken() +
                "&email=" + newAccount.getEmail()); // 본문
        javaMailSender.send(mailMessage);

        return "redirect:/";
    }
}
```
- 실행하면 다음과 같이 토큰이 log에 찍힌 것을 볼 수 있다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_3.jpg"></p>

<br>

## 회원가입 리펙토링과 테스트
- 리팩토링 하기전에 테스트 코드를 먼저 작성하자.
    * 그래야 코드를 변경한 이후에 불안하지 않다.
    * 변경한 코드가 무언가를 깨트리지 않았다는 것을 확인할 수 있다.
<br>

### 목표
- 테스트
    * 폼에 이상한 값이 들어간 경우에 다시 폼이 보여지는지
    * 폼에 값이 정상적인 경우 
        - 가입한 회원 데이터가 존재하는지
        - 이메일이 보내지는지
- 리팩토링
    * 메소드의 길이
    * 코드의 가독성
        - 내가 작성한 코드를 내가 읽기 어렵다면 남들에겐 훨씬 더 어렵다.
    * 코드가 적절한 위치
        - 객체들 사이의 의존 관계
        - 책임이 너무 많진 않은지
<br>

### 구현
- **테스트**
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    ...

    @DisplayName("회원 가입 처리 - 입력값 오류")
    @Test
    void signUpSubmit_with_wrong_input() throws Exception {
        mockMvc.perform(post("/sign-up")
        .param("nickname", "hayoung")
        .param("email", "email..")
        .param("password", "12345"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"));
    }
}
```
- 실행해보면 403에러가 난다.
- 이유 : csrf 토큰이 없기 때문
    * Spring Security를 적용하면 기본적으로 csrf 설정이 활성화 되어있다.
    * csrf 토큰이라는 기술을 자동으로 사용한다.
    * 폼 데이터와 함께 토큰이 서버가 같이 전송되어 토큰을 보고 안전한 데이터인지를 판단한다.
    * SecurityConfig에서 인증이 필요없도록 설정했지만 그렇다고 안전하지 않은 요청을 받아들이는 것은 아니다.
    * 참고) [crsf](https://ko.wikipedia.org/wiki/%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%9A%94%EC%B2%AD_%EC%9C%84%EC%A1%B0)
- 따라서 다음과 같이 crsf 토큰을 추가해줘야 한다.
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    ...

    @DisplayName("회원 가입 처리 - 입력값 오류")
    @Test
    void signUpSubmit_with_wrong_input() throws Exception {
        mockMvc.perform(post("/sign-up")
        .param("nickname", "hayoung")
        .param("email", "email..")
        .param("password", "12345")
        .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"));
    }
}
```
- 옳은 값을 넣었을 경우 테스트(이메일까지)
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private AccountRepository accountRepository;

    @MockBean
    JavaMailSender javaMailSender;

    ...

    @DisplayName("회원 가입 처리 - 입력값 정상")
    @Test
    void signUpSubmit_with_correct_input() throws Exception {
        mockMvc.perform(post("/sign-up")
        .param("nickname", "hayoung")
        .param("email", "hayoung@email.com")
        .param("password", "12345789")
        .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(view().name("redirect:/"));

        assertTrue(accountRepository.existsByEmail("hayoung@email.com"));
        // 이메일을 보내는지 테스트
        then(javaMailSender).should().send(any(SimpleMailMessage.class));
    }
}
```
- **리펙토링**
- AccountController
    * Account 생성과 이메일 보내는 것을 빼서 Service로 만들기
    * 전의 코드와 비교해서 보면 쉽다.
```java
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;
    private final JavaMailSender javaMailSender;

    // Account 생성
    private Account saveNewAccount(@Valid SignUpForm signUpForm) {
        Account account = Account.builder()
                .email(signUpForm.getEmail())
                .nickname(signUpForm.getNickname())
                .password(signUpForm.getPassword()) // TODO encoding 해야한다.
                .studyCreatedByWeb(true)
                .studyEnrollmentResultByWeb(true)
                .studyUpdatedByWeb(true)
                .build();

        return accountRepository.save(account);
    }

    // 이메일 보내기
    private void sendSignUpConfirmEmail(Account newAccount) {
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setTo(newAccount.getEmail());
        mailMessage.setSubject("스터디 올래, 회원 가입 인증"); // 제목
        mailMessage.setText("/check-email-token?token=" + newAccount.getEmailCheckToken() +
                "&email=" + newAccount.getEmail()); // 본문
        javaMailSender.send(mailMessage);
    }

    // 회원가입
    public void processNewAccount(SignUpForm signUpForm) {
        Account newAccount = saveNewAccount(signUpForm);
        newAccount.generateEmailCheckToken();
        sendSignUpConfirmEmail(newAccount);
    }
}
```
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    private final SignUpFormValidator signUpFormValidator;
    private final AccountService accountService;


    // 커스텀 검증
    @InitBinder("signUpForm")
    public void initBinder(WebDataBinder webDataBinder) {
        webDataBinder.addValidators(signUpFormValidator);
    }

    @GetMapping("/sign-up")
    public String signUpForm(Model model) {
        model.addAttribute(new SignUpForm());
        return "account/sign-up";
    }

    @PostMapping("/sign-up")
    public String signUpSubmit(@Valid SignUpForm signUpForm, Errors errors) {
        // JSR 303 검증
        if(errors.hasErrors()) {
            return "account/sign-up";
        }
        // 회원 가입 처리
        accountService.processNewAccount(signUpForm);

        return "redirect:/";
    }
}
```
- 코드를 많이 고쳤음에도 이전에 작성한 테스트를 돌려보면 통과한다.
    * 안전한 변경이었다.
<br>

## 패스워드 인코더
- 절대 입력받은 패스워드를 그대로 평문으로 저장해서는 안된다.
    * Account 엔티티를 저장할 때 패스워드 인코딩을 해야한다.
    * 보통 해싱을 한다.
        - 해싱을 하고 로그인할 때 입력한 패스워드 평문을 해싱해서 일치하는지 확인
        - 양방향으로 암호화-복호화를 할 필요없이 단방향
- 해싱 알고리즘(bcrypt)과 솔트(salt)
    * 해싱 알고리즘을 쓰는 이유
        - 패스워드를 평문으로 저장하면 공격받았을 경우 DB가 털리면 매우 큰 문제가 발생한다.
        - 따라서 패스워드를 해싱해서 DB에 저장해야 한다.
    * 솔트를 쓰는 이유
        - 해싱을 쓸 때 해싱을 해서 DB에 저장하더라도 공격자가 이미 여러개의 문자열들을 해싱해서 그 결과를 비교(dictionary attack)해서 알아낼 수 있다.
        - 따라서 솔트를 추가해서 해싱값이 전혀 다른 값이 나오도록 하면   일치하는 것을 찾는 것이 어렵다. (불가능에 가깝다.)
        - 매번 동일한 salt값을 써야하는 것도 아니고 해싱을 할 때마다 랜덤하게 써도 동작한다.
        - salt값은 인코딩할 때만 사용하고 패스워드로 입력받은 평문과 해싱된 값을 해싱하면 원래 해쉬 값이 나온다.
- 스프링 시큐리티 권장 PasswordEncoder
    * `PasswordEncoderFactories.createDelegatingPasswordEncoder()`
        - 기본적으로 bcrypt를 쓰게 된다.
    * 여러 해시 알고리즘을 지원하는 패스워드 인코더
<br>

### 구현
- AppConfig 생성
```java
@Configuration
public class AppConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```
- AccountService에서 패스워드 인코딩
```java
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;
    private final JavaMailSender javaMailSender;
    private final PasswordEncoder passwordEncoder;

    // Account 생성
    private Account saveNewAccount(@Valid SignUpForm signUpForm) {
        Account account = Account.builder()
                .email(signUpForm.getEmail())
                .nickname(signUpForm.getNickname())
                .password(passwordEncoder.encode(signUpForm.getPassword())) // 인코딩
                .studyCreatedByWeb(true)
                .studyEnrollmentResultByWeb(true)
                .studyUpdatedByWeb(true)
                .build();

        return accountRepository.save(account);
    }

    ...
}
```
- 테스트
    * accountRepository에 `findByEmail()` 메소드 추가
```java
@Transactional(readOnly = true)
public interface AccountRepository extends JpaRepository<Account, Long> {

    boolean existsByEmail(String email);
    boolean existsByNickname(String nickname);

    // 이메일로 account 찾아서 반환
    Account findByEmail(String email);
}
```
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    ...

    @DisplayName("회원 가입 처리 - 입력값 정상")
    @Test
    void signUpSubmit_with_correct_input() throws Exception {
        mockMvc.perform(post("/sign-up")
        .param("nickname", "hayoungㅋ")
        .param("email", "hayoung@email.com")
        .param("password", "12345678")
        .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(view().name("redirect:/"));

        // 패스워드 인코딩 테스트
        Account account = accountRepository.findByEmail("hayoung@email.com");
        assertNotNull(account);
        assertNotEquals(account.getPassword(), "12345678");

        then(javaMailSender).should().send(any(SimpleMailMessage.class));
    }
}
```
<br>

## 인증 메일 확인

### 목표
- GET “/check-email-token” token=${token} email=${email} 요청 처리
    * 이메일이 정확하지 않은 경우에 대한 에러 처리
    * 토큰이 정확하지 않은 경우에 대한 에러 처리
    * 이메일과 토큰이 정확한 경우 가입 완료 처리
        - 가입 일시 설정
        - 이메일 인증 여부 true로 설정
- 인증 확인 뷰
    * 입력값에 오류가 있는 경우 적절한 메시지 출력.
    * 인증이 완료된 경우, 환영 문구와 함께 몇번째 사용자인지 보여주기.
<br>

### 구현
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    private final SignUpFormValidator signUpFormValidator;
    private final AccountService accountService;
    private final AccountRepository accountRepository;

    ...

    @GetMapping("/check-email-token")
    public String checkEmailToken(String token, String email, Model model) {
        Account account = accountRepository.findByEmail(email);
        String view = "account/checked-email";
        
        // account가 없을 경우
        if (account == null) {
            model.addAttribute("error", "wrong.email");
            return view;
        }
        // 토큰이 일치하지 않을 경우
        if (!account.getEmailCheckToken().equals(token)) {
            model.addAttribute("error", "wrong.token");
            return view;
        }

        account.setEmailVerified(true);
        account.setJoinedAt(LocalDateTime.now());
        model.addAttribute("numberOfUser", accountRepository.count()); // count()는 기본으로 제공하는 기능이다.
        model.addAttribute("nickname", account.getNickname());
        return view;
    }
}
```
- 인증 확인 뷰 작성
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>StudyOlle</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.5.3/dist/css/bootstrap.min.css" integrity="sha384-TX8t27EcRE3e/ihU7zmQxVncDAy5uIKz4rEkgIXeMed4M0jlfIDPvg6uqKI2xXr2" crossorigin="anonymous">
    <style>
        .container {
            max-width: 100%;
        }
    </style>
</head>

<body class="bg-light">
    <nav class="navbar navbar-expand-sm navbar-dark bg-dark">
        <a class="navbar-brand" href="/" th:href="@{/}">
            <img src="/images/logo_sm.png" width="30" height="30">
        </a>
        <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav mr-auto">
                <li class="nav-item">
                    <form th:action="@{/search/study}" class="form-inline" method="get">
                        <input class="form-control mr-sm-2" name="keyword" type="search" placeholder="스터디 찾기" aria-label="Search" />
                    </form>
                </li>
            </ul>

            <ul class="navbar-nav justify-content-end">
                <li class="nav-item">
                    <a class="nav-link" href="#" th:href="@{/login}">로그인</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="#" th:href="@{/signup}">가입</a>
                </li>
            </ul>
        </div>
    </nav>

    <div class="py-5 text-center" th:if="${error}">
        <p class="lead">스터디올래 이메일 확인</p>
        <div class="alert alert-danger" role="alert">
            이메일 확인 링크가 정확하지 않습니다.
        </div>
    </div>

    <div class="py-5 text-center" th:if="${error == null}">
        <p class="lead">스터디올래 이메일 확인</p>
        <h2>
            이메일을 확인했습니다. <span th:text="${numberOfUser}">10</span>번째 회원,
            <span th:text="${nickname}">김하영</span>님 가입을 축하합니다.
        </h2>
        <small class="text-info">이제부터 가입할 때 사용한 이메일 또는 닉네임과 패스워드로 로그인 할 수 있습니다.</small>
    </div>
</body>
</html>
```
- 실행해서 가입하고 이메일 인증을 하면 에러가 난다.
    * account의 emailCheckToken이 null이다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_4.jpg"></p>

- 트랜잭션이 없어서 발생하는 에러이다.
    * AccountService에서 만든 newAccount객체는 detached객체이다.
    * 따라서 DB에 동기화가 되지 않고 emailCheckToken값이 DB에 저장되지 않았다.
    * `saveNewAccount()`에서 `save()`에 의해 저장되고 트랜잭션 안이지만 나오면 트랜잭션 범위를 벗어나 detached 상태이다.
    * 따라서 `processNewAccount()`에 `@Transactional`을 붙여주면 persist상태가 되서 저장이 된다.
    * 참고로 테스트를 작성할 때 emailCheckToken이 생성됐는지 확인했어야 하지만 놓쳤다.
        - 테스트를 완벽히 신뢰할 수 없는 이유
```java
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;
    private final JavaMailSender javaMailSender;
    private final PasswordEncoder passwordEncoder;

    ...

    // 회원가입
    @Transactional
    public void processNewAccount(SignUpForm signUpForm) {
        Account newAccount = saveNewAccount(signUpForm);
        newAccount.generateEmailCheckToken();
        sendSignUpConfirmEmail(newAccount);
    }
}
```
- 이제 다시 시도하면 다음과 같이 결과가 잘 나온다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_5.jpg"></p>

<br>

## 인증 메일 확인 테스트 및 리팩토링

### 목표
- 테스트
    * 입력값이 잘못 된 경우
        - error 프로퍼티가 model에 들어있는지 확인
        - 뷰 이름이 account/checkd-email인지 확인
    * 입력값이 올바른 경우
        - 모델에 error가 없는지 확인
        - 모델에 numberOfUser가 있는지 확인
        - 모델에 nickname이 있는지 확인
        - 뷰 이름 확인
- 리팩토링
    * 코드의 위치 조절
<br>

### 구현
- 테스트 코드 작성
    * 테스트도 트랜잭션이 없기 때문에 `@Transactional`어노테이션을 붙여줘야 한다.
```java
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    ...

    @DisplayName("인증 메일 확인 - 입력값 오류")
    @Test
    void checkEmailToken_with_wrong_input() throws Exception {
        mockMvc.perform(get("/check-email-token")
                .param("token", "jacoiuahnc")
                .param("email", "email@email.com"))
                .andExpect(status().isOk()) // 상태는 isOk이지만
                .andExpect(model().attributeExists("error")) // model에는 error가 있는지 확인
                .andExpect(view().name("account/checked-email")); // view 이름이 account/checkd-email인지 확인
    }

    @DisplayName("인증 메일 확인 - 입력값 정상")
    @Test
    void checkEmailToken() throws Exception {
        Account account = Account.builder()
                .email("test@email.com")
                .password("12345678")
                .nickname("hayoung")
                .build();
        Account newAccount = accountRepository.save(account);
        newAccount.generateEmailCheckToken();

        mockMvc.perform(get("/check-email-token")
                .param("token", newAccount.getEmailCheckToken())
                .param("email", newAccount.getEmail()))
                .andExpect(status().isOk())
                .andExpect(model().attributeDoesNotExist("error")) // model에 error가 없고
                .andExpect(model().attributeExists("nickname")) // nickname과
                .andExpect(model().attributeExists("numberOfUser")) // numberOfUser는 있고
                .andExpect(view().name("account/checked-email")); // view 이름은 동일
    }
    
    ...
}
```
- 리팩토링
    * AccountController에서 이메일 인증과 가입날짜 저장을 메소드로 분리
```java
@Entity
@Getter @Setter @EqualsAndHashCode(of = "id")
@Builder @AllArgsConstructor @NoArgsConstructor
public class Account {

    ...

    public void completeSignUp() {
        this.emailVerified = true;
        this.joinedAt = LocalDateTime.now();
    }
}
```
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    ...

    @GetMapping("/check-email-token")
    public String checkEmailToken(String token, String email, Model model) {
        
        ...

        // 메소드로 분리
        account.completeSignUp();

        ...
    }
}
```
- 테스트 코드를 다시 실행해 확인하면 테스트 코드가 정상적으로 실행된다.
<br>

## 가입 완료 후 자동 로그인

### 목표
- 회원 가입 완료시 자동 로그인
- 이메일 인증 완료시 자동 로그인
<br>

### 구현
- AccountService에 `login()` 메소드 생성
    * 원래는 폼에서 입력받는 username과 plain text인 password를 가지고 AuthenticationManager를 통해서 인증해야 한다.
    * 그러나 인코딩한 password밖에 접근을 하지 못하기 때문에 다른 방법을 사용한다.
```java
@Service
@RequiredArgsConstructor
public class AccountService {

    ...

    public void login(Account account) {
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(
                account.getNickname(),
                account.getPassword(),
                List.of(new SimpleGrantedAuthority("ROLE_USER")));
        SecurityContextHolder.getContext().setAuthentication(token);

//        정석적인 방법
//        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(username, password);
//        Authentication authentication = authenticationManager.authenticate(token);
//        SecurityContext context = SecurityContextHolder.getContext();
//        context.setAuthentication(authentication);

    }
}
```
- AccountController에 자동 로그인 코드 추가
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    ...

    @PostMapping("/sign-up")
    public String signUpSubmit(@Valid SignUpForm signUpForm, Errors errors) {
        // JSR 303 검증
        if (errors.hasErrors()) {
            return "account/sign-up";
        }
        // 회원 가입 처리
        Account account = accountService.processNewAccount(signUpForm);
        accountService.login(account); // 회원 가입이 완료됐을 경우 자동으로 로그인

        return "redirect:/";
    }

    @GetMapping("/check-email-token")
    public String checkEmailToken(String token, String email, Model model) {
        Account account = accountRepository.findByEmail(email);
        String view = "account/checked-email";
        
        // account가 없을 경우
        if (account == null) {
            model.addAttribute("error", "wrong.email");
            return view;
        }
        // 토큰이 일치하지 않을 경우
        if (account.isValidToken(token)) {
            model.addAttribute("error", "wrong.token");
            return view;
        }

        account.completeSignUp();
        accountService.login(account); // token 인증시에도 자동 로그인
        model.addAttribute("numberOfUser", accountRepository.count());
        model.addAttribute("nickname", account.getNickname());
        return view;
    }
}
```
- 테스트
    * 스프링부트와 스프링 시큐리티가 연동되어 있으면 MockMvc에 여러가지 기능을 추가로 사용할 수 있다.
        - 스프링 시큐리티가 있는 MockMvc와 없는 MockMvc는 다르다.
        - csrf가 그러한 기능 중 하나이다.
        - 인증이 됐는지 안됐는지 판단하는 것도 기능에 있다.
```java
@Transactional
@SpringBootTest
@AutoConfigureMockMvc
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private AccountRepository accountRepository;

    @MockBean
    JavaMailSender javaMailSender;

    @DisplayName("인증 메일 확인 - 입력값 오류")
    @Test
    void checkEmailToken_with_wrong_input() throws Exception {
        mockMvc.perform(get("/check-email-token")
                .param("token", "jacoiuahnc")
                .param("email", "email@email.com"))
                .andExpect(status().isOk())
                .andExpect(model().attributeExists("error"))
                .andExpect(view().name("account/checked-email"))
                .andExpect(unauthenticated());

    }

    @DisplayName("인증 메일 확인 - 입력값 정상")
    @Test
    void checkEmailToken() throws Exception {
        Account account = Account.builder()
                .email("test@email.com")
                .password("12345678")
                .nickname("hayoung")
                .build();
        Account newAccount = accountRepository.save(account);
        newAccount.generateEmailCheckToken();

        mockMvc.perform(get("/check-email-token")
                .param("token", newAccount.getEmailCheckToken())
                .param("email", newAccount.getEmail()))
                .andExpect(status().isOk())
                .andExpect(model().attributeDoesNotExist("error"))
                .andExpect(model().attributeExists("nickname"))
                .andExpect(model().attributeExists("numberOfUser"))
                .andExpect(view().name("account/checked-email"))
                .andExpect(authenticated().withUsername("hayoung"));
    }

    @DisplayName("회원 가입 화면이 보이는지 테스트")
    @Test
    void signUpForm() throws Exception {
        mockMvc.perform(get("/sign-up"))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(model().attributeExists("signUpForm"))
                .andExpect(unauthenticated());

    }

    @DisplayName("회원 가입 처리 - 입력값 오류")
    @Test
    void signUpSubmit_with_wrong_input() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("nickname", "hayoung")
                .param("email", "email..")
                .param("password", "12345")
                .with(csrf()))
                .andExpect(status().isOk())
                .andExpect(view().name("account/sign-up"))
                .andExpect(unauthenticated());
    }

    @DisplayName("회원 가입 처리 - 입력값 정상")
    @Test
    void signUpSubmit_with_correct_input() throws Exception {
        mockMvc.perform(post("/sign-up")
                .param("nickname", "hayoung")
                .param("email", "hayoung@email.com")
                .param("password", "12345678")
                .with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(view().name("redirect:/"))
                .andExpect(authenticated().withUsername("hayoung"));

        // 패스워드 인코딩 테스트
        Account account = accountRepository.findByEmail("hayoung@email.com");
        assertNotNull(account);
        assertNotEquals(account.getPassword(), "12345678");
        assertNotNull(account.getEmailCheckToken());
        then(javaMailSender).should().send(any(SimpleMailMessage.class));
    }
}
```
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_6.jpg"></p>

<br>

## 메인 네비게이션 메뉴 변경

### 목표
- 네비게이션 뷰
    * 인증 정보가 없는 경우
    * 인증 정보가 있는 경우

### 구현
- 타임리프 스프링 시큐리티 의존성 추가
    * 뷰에서 인증 정보를 참조할 때 유용한 라이브러리
```xml
<dependency>
	<groupId>org.thymeleaf.extras</groupId>
	<artifactId>thymeleaf-extras-springsecurity5</artifactId>
</dependency>
```
- index.html 작성
```html
<!DOCTYPE html>
<html lang="en"
      xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
<head>
    <meta charset="UTF-8">
    <title>StudyOlle</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.5.3/dist/css/bootstrap.min.css" integrity="sha384-TX8t27EcRE3e/ihU7zmQxVncDAy5uIKz4rEkgIXeMed4M0jlfIDPvg6uqKI2xXr2" crossorigin="anonymous">
    <style>
        .container {
            max-width: 100%;
        }
    </style>
</head>

<body class="bg-light">
<nav class="navbar navbar-expand-sm navbar-dark bg-dark">
    <a class="navbar-brand" href="/" th:href="@{/}">
        <img src="/images/logo_sm.png" width="30" height="30">
    </a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
    </button>

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
        <ul class="navbar-nav mr-auto">
            <li class="nav-item">
                <form th:action="@{/search/study}" class="form-inline" method="get">
                    <input class="form-control mr-sm-2" name="keyword" type="search" placeholder="스터디 찾기" aria-label="Search" />
                </form>
            </li>
        </ul>

        <ul class="navbar-nav justify-content-end">
            <li class="nav-item" sec:authorize="!isAuthenticated()">
                <a class="nav-link" th:href="@{/login}">로그인</a>
            </li>
            <li class="nav-item" sec:authorize="!isAuthenticated()">
                <a class="nav-link" th:href="@{/sign-up}">가입</a>
            </li>
            </li>
            <li class="nav-item" sec:authorize="isAuthenticated()">
                <a class="nav-link btn btn-outline-primary" th:href="@{/notifications}">스터디 개설</a>
            </li>
            <li class="nav-item dropdown" sec:authorize="isAuthenticated()">
                <a class="nav-link dropdown-toggle" href="#" id="userDropdown" role="button" data-toggle="dropdown"
                   aria-haspopup="true" aria-expanded="false">
                    프로필
                </a>
                <div class="dropdown-menu dropdown-menu-sm-right" aria-labelledby="userDropdown">
                    <h6 class="dropdown-header">
                        <span sec:authentication="name">Username</span>
                    </h6>
                    <a class="dropdown-item" th:href="@{'/profile/' + ${#authentication.name}}">프로필</a>
                    <a class="dropdown-item" >스터디</a>
                    <div class="dropdown-divider"></div>
                    <a class="dropdown-item" href="#" th:href="@{'/settings/profile'}">설정</a>
                    <form class="form-inline my-2 my-lg-0" action="#" th:action="@{/logout}" method="post">
                        <button class="dropdown-item" type="submit">로그아웃</button>
                    </form>
                </div>
            </li>
        </ul>
    </div>
</nav>

<div class="container">
    <div class="py-5 text-center">
        <h2>스터디올래</h2>
    </div>

    <footer th:fragment="footer">
        <div class="row justify-content-center">
            <img class="mb-2" src="/images/logo_long_kr.png" alt="" width="100">
            <small class="d-block mb-3 text-muted">&copy; 2020</small>
        </div>
    </footer>
</div>
<script src="https://code.jquery.com/jquery-3.5.1.slim.min.js" integrity="sha384-DfXdz2htPH0lsSSs5nCTpuj/zy4C+OGpamoFVy38MVBnE+IbbVYUew+OrCXaRkfj" crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.1/dist/umd/popper.min.js" integrity="sha384-9/reFTGAW83EW2RDu2S0VKaIzap3H66lZH81PoYlFhbGU+6BZp6G7niu735Sk7lN" crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@4.5.3/dist/js/bootstrap.min.js" integrity="sha384-w1Q4orYjBQndcko6MimVbzY0tgp4pWB4lZ7lr30WKz0vr/aWKhXdBNmNb5D92v7s" crossorigin="anonymous"></script>
<script type="application/javascript">
        (function () {

        }())
    </script>
</body>
</html>
```
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_7.jpg"></p>

<br>

## 프론트엔드 라이브러리 설정
- 프론트엔드 라이브러리를 cdn으로 받아오는 것도 나쁘지 않지만 프로젝트안에 패키지화 시킬 수도 있다.
- 크게 2가지 방법이 있다. 
    * WebJar vs NPM
        - WebJar보다는 **NPM** 선호.
        - WebJar는 라이브러리 업데이트가 느리고 제공하지 않는 라이브러리도 많다.
    * 직접 다운로드 받아서 resources/static 아래에 모아놓을 수도 있지만 3rd-party 라이브러리 관리하는 방법으로는 적절하지 않다.
        * 3rd-party 라이브러리는 업데이트 될 때마다 dependency을 쉽게 관리할 수 있어야 한다.
        * 의존성 관리 툴에 지원을 받아서 사용하는 것이 좋다.
<br>

### 스프링부트와 NPM
- src/main/resources/static 디렉토리 이하는 정적 리소스로 제공한다.
- package.json에 프론트엔드 라이브러리를 제공한다.
- 이 둘을 응용하면, 즉 static 디렉토리 아래에 package.json을 사용해서 라이브러리를 받아오면 정적 리소스로 프론트엔드 라이브러리를 사용할 수 있다.
<br>

### NPM 설치와 라이브러리 추가
- [NPM 설치](https://www.npmjs.com/get-npm)
- resources/static에 package.json을 만든다.
    * `npm init`
    * 설치하면 다음과 같이 resources/static에 package.json이 생긴 것을 볼 수 있다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_8.jpg"></p>

- 라이브러리 설치
    * `npm install bootstrap`
    * `npm install jquery --save`
    * 설치 후 package.json의 dependencies에 라이브러리가 만들어져 있다.
    * 참고) package.lock.json은 라이브러리의 버전을 고정시키는 것이다.
        - 의존성 트리에 대한 정보를 가지고 있으며 package-lock.json 파일이 작성된 시점의 의존성 트리가 다시 생성될 수 있도록 보장한다.
<br>

### 구현
- 이제 프론트엔드 라이브러리를 cdn에서 받아오지 않아도 된다.
```html
<link rel="stylesheet" href="/node_modules/bootstrap/dist/css/bootstrap.min.css" />

    ...

<script src="/node_modules/jquery/dist/jquery.min.js"></script>
<script src="/node_modules/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
```
<br>

### 고려해야 할 점
- 버전관리
    * 자동으로 받아오는 라이브러리들을 ignore 시킨다.
```gitignore

...

### NPM ###
src/main/resources/static/node_modules
src/main/resources/static/node
```
- 빌드
    * 메이븐 pom.xml을 빌드할 때 static 디렉토리 아래에 있는 package.json도 빌드하도록 설정해야 한다.
        - 프론트쪽 모듈도 빌드해야 한다.
    * 빌드를 안하면 프론트엔드 라이브러리를 받아오지 않아서 뷰에서 필요한 참조가 끊어지고 화면이 제대로 보이지 않는다.
    * 여러가지 maven 관련 플러그인 중 프론트엔드 모듈을 실행해주는 플러그인을 추가한다.
    * 추가 후 `/mvnw test`로 테스트 해보면 정상적으로 동작한다.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.3.4.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.studyolle</groupId>
	<artifactId>studyolle</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>studyolle</name>
	<description>Study management web service</description>

	<properties>
		<java.version>11</java.version>
	</properties>

	<dependencies>

		...

	</dependencies>

	<build>
		<plugins>
			
            ... 
            
			<plugin>
				<groupId>com.github.eirslett</groupId>
				<artifactId>frontend-maven-plugin</artifactId>
				<version>1.8.0</version>
				<configuration>
					<nodeVersion>v4.6.0</nodeVersion>
					<workingDirectory>src/main/resources/static</workingDirectory>
				</configuration>
				<executions>
					<execution>
						<id>install node and npm</id>
						<goals>
							<goal>install-node-and-npm</goal>
						</goals>
						<phase>generate-resources</phase>
					</execution>
					<execution>
						<id>npm install</id>
						<goals>
							<goal>npm</goal>
						</goals>
						<phase>generate-resources</phase>
						<configuration>
							<arguments>install</arguments>
						</configuration>
					</execution>
				</executions>
			</plugin>

		</plugins>
	</build>
</project>
```
- Spring Security 설정
    * Spring Security를 사용하면 기본적으로 모든 요청에 보안을 적용하지만 SecurityConfig에 예외를 설정했었다.
    * SecurityConfig클래스의 `configure()`에서 흔히 사용하는 위치에 스프링 시큐리티 필터를 사용하지 않도록 했었다.
    * 프론트엔드 라이브러리에서 요청이 /node_modules로 시작하기 때문에 이것도 추가해줘야 한다. 
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    ...

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring()
                .mvcMatchers("/node_modules/**") // 경로 추가
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }
}
```
- 위의 3가지 고려사항을 수정하고 실행하면 다음과 같이 프론트엔드 라이브러리 설정 변경 후에도 정상적으로 실행되는 것을 확인할 수 있다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_9.jpg"></p>

<br>

## 뷰 중복 코드 제거
- 타임리프의 Fragment를 사용해서 뷰의 중복 코드를 제거한다.
    * [참고](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html#including-template-fragments)
    * Fragment 정의 : `th:fragment`
    * Fragment 사용 : `th:insert`, `th:replace`
- header, footer와 네이게이션바를 재사용
    * fragments.html을 생성해서 fragment들을 작성하고 html 파일들에 적용한다.
```html
<!DOCTYPE html>
<html lang="en"
      xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">

<head th:fragment="head">
    <meta charset="UTF-8">
    <title>StudyOlle</title>
    <link rel="stylesheet" href="/node_modules/bootstrap/dist/css/bootstrap.min.css" />
    <style>
        .container {
            max-width: 100%;
        }
    </style>
</head>

<nav th:fragment="main-nav" class="navbar navbar-expand-sm navbar-dark bg-dark">
    <a class="navbar-brand" href="/" th:href="@{/}">
        <img src="/images/logo_sm.png" width="30" height="30">
    </a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
    </button>

    <div class="collapse navbar-collapse" id="navbarSupportedContent">
        <ul class="navbar-nav mr-auto">
            <li class="nav-item">
                <form th:action="@{/search/study}" class="form-inline" method="get">
                    <input class="form-control mr-sm-2" name="keyword" type="search" placeholder="스터디 찾기" aria-label="Search" />
                </form>
            </li>
        </ul>

        <ul class="navbar-nav justify-content-end">
            <li class="nav-item" sec:authorize="!isAuthenticated()">
                <a class="nav-link" th:href="@{/login}">로그인</a>
            </li>
            <li class="nav-item" sec:authorize="!isAuthenticated()">
                <a class="nav-link" th:href="@{/sign-up}">가입</a>
            </li>
            </li>
            <li class="nav-item" sec:authorize="isAuthenticated()">
                <a class="nav-link btn btn-outline-primary" th:href="@{/notifications}">스터디 개설</a>
            </li>
            <li class="nav-item dropdown" sec:authorize="isAuthenticated()">
                <a class="nav-link dropdown-toggle" href="#" id="userDropdown" role="button" data-toggle="dropdown"
                   aria-haspopup="true" aria-expanded="false">
                    프로필
                </a>
                <div class="dropdown-menu dropdown-menu-sm-right" aria-labelledby="userDropdown">
                    <h6 class="dropdown-header">
                        <span sec:authentication="name">Username</span>
                    </h6>
                    <a class="dropdown-item" th:href="@{'/profile/' + ${#authentication.name}}">프로필</a>
                    <a class="dropdown-item" >스터디</a>
                    <div class="dropdown-divider"></div>
                    <a class="dropdown-item" href="#" th:href="@{'/settings/profile'}">설정</a>
                    <form class="form-inline my-2 my-lg-0" action="#" th:action="@{/logout}" method="post">
                        <button class="dropdown-item" type="submit">로그아웃</button>
                    </form>
                </div>
            </li>
        </ul>
    </div>
</nav>

<footer th:fragment="footer">
    <div class="row justify-content-center">
        <img class="mb-2" src="/images/logo_long_kr.png" alt="" width="100">
        <small class="d-block mb-3 text-muted">&copy; 2020</small>
    </div>
</footer>

</html>
```
```html
    ...

<head th:replace="fragments.html :: head"></head>

    ...

<div th:replace="fragments.html :: main-nav"></div>
    
    ...

<div th:replace="fragments.html :: footer"></div>
```
- 실행해보면 정상적으로 실행된다.
<br>

## 첫 페이지 보완

### 목표
- 네비게이션 바에 [Fontawesome](https://fontawesome.com/)으로 아이콘 추가
- 이메일 인증을 마치지 않은 사용자에게 메시지 보여주기
- [jdenticon](https://jdenticon.com/)으로 프로필 기본 이미지 생성하기
<br>

### 구현
- NPM으로 프론트엔트 라이브러리 설치
    * `npm install font-awesome`
        - 아이콘을 제공해주는 라이브러리
    * `npm install jdenticon`
        - 특정한 문자열에 해당하는 아바타를 만들어주는 라이브러리
- fontawesome 아이콘 사용하기
    * fragments.html에 fontawesome 추가
    * 벨 모양 알림 아이콘 추가
    * 스터디 개설에 \+ 모양 아이콘 추가
    * 다른 아이콘을 찾고 싶으면 node_modules/font-awesome/README.md에 있는 사이트에서 검색하면 된다.
```html
    ...
<link rel="stylesheet" href="/node_modules/font-awesome/css/font-awesome.min.css" />
    
    ...

<li class="nav-item" sec:authorize="isAuthenticated()">
    <a class="nav-link" th:href="@{/notifications}">
        <i class="fa fa-bell-o" aria-hidden="true"></i>
    </a>
</li>
<li class="nav-item" sec:authorize="isAuthenticated()">
    <a class="nav-link btn btn-outline-primary" th:href="@{/notifications}">
        <i class="fa fa-plus" aria-hidden="true"></i> 스터디 개설
    </a>
</li>

    ...
```
- Jdenticon으로 프로필 기본 이미지 생성하기
    * fragments.html에 Jdenticon추가
    * 참고) bootstrap과 jquery는 대부분의 뷰에서 참조할 것이기 때문에 head에 넣어준다.
```html
    ...

<script src="/node_modules/jquery/dist/jquery.min.js"></script>
<script src="/node_modules/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
<script src="/node_modules/jdenticon/dist/jdenticon.min.js"></script>

    ...
    <li class="nav-item dropdown" sec:authorize="isAuthenticated()">
                <a class="nav-link dropdown-toggle" href="#" id="userDropdown" role="button" data-toggle="dropdown"
                   aria-haspopup="true" aria-expanded="false">
                    <svg data-jdenticon-value="user127" th:data-jdenticon-value="${#authentication.name}"
                         width="24" height="24" class="rounded border bg-light"></svg>
                </a>
                
                ...
```
- 이메일 인증을 마치지 않은 사용자에게 메시지 보여주기
    * bootstrap 경고창
    * fragments.html이 아닌 첫 페이지(index.html)에서만 보여주도록 설정한다.
```html
    ...

<body class="bg-light">
    <div th:replace="fragments.html :: main-nav"></div>
    <div class="alert alert-warning" role="alert" th:if="${account != null && !account.emailVerified}">
        스터디올래 가입을 완료하려면 <a href="#" th:href="@{/check-email}" class="alert-link">계정 인증 이메일을 확인</a>하세요.
    </div>

    ...
```
<br>

## 현재 인증된 사용자 정보 참조
- `@AuthenticationPrincipal`
    * Spring Security가 지원하는 스프링 웹 MVC 기능 중 하나로 인증된 사용자 정보를 참조하는 방법이다.
    * 핸들러 매개변수로 현재 인증된 **Principal**을 참조할 수 있다.
        - Principal은 인증할 때 Authentication에 들어있는 첫 번째 파라미터이다.
        - AccountService의 `login()`에서 `account.getNickname()`
    * SpEL을 사용해서 Principal 내부 정보에 접근할 수도 있다.
        - `@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : account")`
        - 익명 인증인 경우에는 null로 설정하고, 아닌 경우에는 account 프로퍼티를 조회해서 설정
<br>

### 구현
- mainController 생성
```java
@Controller
public class MainController {

    @GetMapping("/")
    public String home(@CurrentUser Account account, Model model) {
        if (account != null) { // 인증을 한 사용자의 경우
            model.addAttribute(account);
        }

        return "index";
    }

}
```
- CurrentUser 어노테이션 생성
    * `@Retention`
        - 이 어노테이션 정보를 언제까지 유지할 것인가 
        - Runtime까지 유지하겠다는 뜻
    * `@Target`
        - 이 어노테이션을 파라미터에만 붙일 수 있도록 설정
    * `@AuthenticationPrincipal`
        - 익명 인증인 경우에는 null로 설정하고, 아닌 경우에는 account 프로퍼티를 조회해서 설정
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : account")
public @interface CurrentUser {
    
}
```
- UserAccount 생성
    * account라는 프로퍼티를 들고 있는 중간 역할을 해주는 객체
    * Spring Security가 다루는 user정보와 도메인에서 다루는 user정보를 이어주는 역할
```java
@Getter
public class UserAccount extends User {

    private Account account;

    public UserAccount(Account account) {
        super(account.getNickname(), account.getPassword(), List.of(new SimpleGrantedAuthority("ROLE_USER")));
        this.account = account;
    }
}
``` 
- 실행하면 뷰에 전달된 account가 null이므로 경고창이 뜨지 않고 가입을 하면 경고창이 나온다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_10.jpg"></p>

- 이미 구현한 이메일 인증을 하면 다음과 같이 이메일 인증이 되고 홈으로 돌아가도 인증을 하라는 경고창이 뜨지 않는다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_11.jpg"></p>

<br>

## 가입 확인 이메일 재전송

### 목표
- 가입 확인 이메일을 재전송할 수 있는 기능 구현
- 하지만, 너무 자주 이메일을 전송할 경우 리소스를 낭비할 수 있다는 문제가 있다.
- 보완책으로, 1시간에 한 번만 인증 메일을 전송할 수 있도록 제한한다.
<br>

### 구현
- 이메일 재전송 뷰 생성
    * check-email.html
        - 에러메세지가 있을 때 없을 떄 나눠서 보여준다.
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head th:replace="fragments.html :: head"></head>

<body class="bg-light">
<nav th:replace="fragments.html :: main-nav"></nav>

<div class="container">
    <div class="py-5 text-center" th:if="${error != null}">
        <p class="lead">스터디올래 가입</p>
        <div  class="alert alert-danger" role="alert" th:text="${error}"></div>
        <p class="lead" th:text="${email}">your@email.com</p>
    </div>

    <div class="py-5 text-center" th:if="${error == null}">
        <p class="lead">스터디올래 가입</p>

        <h2>스터디올래 서비스를 사용하려면 인증 이메일을 확인하세요.</h2>

        <div>
            <p class="lead" th:text="${email}">your@email.com</p>
            <a class="btn btn-outline-info" th:href="@{/resend-confirm-email}">인증 이메일 다시 보내기</a>
        </div>
    </div>
</div>
</body>
</html>
```
- Account 엔티티에 이메일 인증 토큰 생성 시간 필드와 1시간이 지나서 보낼 수 있는지 확인하는 메서드 추가
```java
@Entity
@Getter @Setter @EqualsAndHashCode(of = "id")
@Builder @AllArgsConstructor @NoArgsConstructor
public class Account {

    ...

    private LocalDateTime emailCheckTokenGeneratedAt;

    ...

    public boolean canSendConfirmEmail() {
        return this.emailCheckTokenGeneratedAt.isBefore(LocalDateTime.now().minusHours(1));
    }
}
```
- 이메일 인증을 할 수 있게 하기 위해 accountService에 `sendSignUpConfirmEmail()`을 public으로 수정
```java
@Service
@RequiredArgsConstructor
public class AccountService {

    ...

    // public으로 수정
    public void sendSignUpConfirmEmail(Account newAccount) {
        SimpleMailMessage mailMessage = new SimpleMailMessage();
        mailMessage.setTo(newAccount.getEmail());
        mailMessage.setSubject("스터디 올래, 회원 가입 인증"); // 제목
        mailMessage.setText("/check-email-token?token=" + newAccount.getEmailCheckToken() +
                "&email=" + newAccount.getEmail()); // 본문
        javaMailSender.send(mailMessage);
    }

    ...
}
```
- AccountController에 맵핑 추가
    * `return "redirect:/";`
        - redirect하지 않으면 화면을 refresh할 때마다 이메일을 보낸다. 따라서 홈 화면으로 redirect시킨다.
        - 참고) 회원가입도 마찬가지였다.
        - 요청을 처리했을 때 다시 요청되지 않도록 적절한 페이지로 redirect시키는 것을 고려해보면 좋다.
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    ...

    @GetMapping("/check-email")
    public String checkEmail(@CurrentUser Account account, Model model) {
        model.addAttribute("email", account.getEmail());
        return "account/check-email";
    }

    @GetMapping("/resend-confirm-email")
    public String resendConfirmEmail(@CurrentUser Account account, Model model) {
        if (!account.canSendConfirmEmail()) {
            model.addAttribute("error", "인증 이메일은 1시간에 한번만 전송할 수 있습니다.");
            model.addAttribute("email", account.getEmail());
            return "account/check-email";
        }

        accountService.sendSignUpConfirmEmail(account);
        return "redirect:/";
    }
}
```
- 실행하면 다음과 같이 이메일 인증 화면이 정상적으로 나오고 인증 이메일을 다시 보내면 1시간에 1번만 보낼 수 있다고 경고 메세지가 나온다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_12.jpg"></p>

<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_13.jpg"></p>

<br>

## 로그인&로그아웃

### 목표
- Spring Security를 사용해 커스텀 로그인 페이지 만들기
- 로그아웃 기능 구현하기
<br>

### 구현
- SecurityConfig에 로그인, 로그아웃 설정 추가
    * `http.formLogin()`
        - formLogin 기능을 사용할 수 있다.
        - `logPage()`를 사용해서 커스텀한 로그인 페이지를 사용할 수 있다. 
        - 그냥 formLogin만 사용하면 Spring Security가 기본적으로 제공하는 로그인 페이지가 나온다.
    * `http.logout()`
        - 기본적으로 켜져있긴 하지만 로그아웃이 성공했을 때 특정한 작업을 수행하도록 할 수 있다.
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        
        ...
        
        http.formLogin()
                .loginPage("/login").permitAll();
        http.logout()
                .logoutSuccessUrl("/");
    }

    ...
}
```
- 로그인 요청 맵핑 추가
```java
@Controller
public class MainController {

    ...

    @GetMapping("/login")
    public String login() {
        return "login";
    }
}
```
- 로그인 페이지 뷰 생성
```html
<!DOCTYPE html>
<html lang="en"
      xmlns:th="http://www.thymeleaf.org">
<head th:replace="fragments.html :: head"></head>

<body class="bg-light">
    <div th:replace="fragments.html :: main-nav"></div>

    <div class="container">
        <div class="py-5 text-center">
            <p class="lead">스터디올래</p>
            <h2>로그인</h2>
        </div>
        <div class="row justify-content-center">
            <div th:if="${param.error}" class="alert alert-danger" role="alert">
                <p>이메일(또는 닉네임)과 패스워드가 정확하지 않습니다.</p>
                <p>또는 확인되지 않은 이메일을 사용했습니다. 이메일을 확인해 주세요.</p>
                <p>
                    확인 후 다시 입력하시거나, <a href="#" th:href="@{/find-passsword}">패스워드 찾기</a>를 이용하세요.
                </p>
            </div>

            <form class="needs-validation col-sm-6" action="#" th:action="@{/login}" method="post" novalidate>
                <div class="form-group">
                    <label for="username">이메일 또는 닉네임</label>
                    <input id="username" type="text" name="username" class="form-control"
                           placeholder="your@email.com" aria-describedby="emailHelp" required>
                    <small id="emailHelp" class="form-text text-muted">
                        가입할 때 사용한 이메일 또는 닉네임을 입력하세요.
                    </small>
                    <small class="invalid-feedback">이메일을 입력하세요.</small>
                </div>
                <div class="form-group">
                    <label for="password">패스워드</label>
                    <input id="password" type="password" name="password" class="form-control"
                           aria-describedby="passwordHelp" required>
                    <small id="passwordHelp" class="form-text text-muted">
                        패스워드가 기억나지 않는다면, <a href="#" th:href="@{/emaillogin}">패스워드 없이 로그인하기</a>
                    </small>
                    <small class="invalid-feedback">패스워드를 입력하세요.</small>
                </div>

                <div class="form-group">
                    <button class="btn btn-success btn-block" type="submit"
                            aria-describedby="submitHelp">로그인</button>
                    <small id="submitHelp" class="form-text text-muted">
                        스터디올래에 처음 오신거라면 <a href="#" th:href="@{/signup}">계정을 먼저 만드세요.</a>
                    </small>
                </div>
            </form>
        </div>

        <div th:replace="fragments.html :: footer"></div>
    </div>
    <script th:replace="fragments.html :: form-validation"></script>

</body>
</html>
```
- UserDetailsService 구현
    * post로 가는 로그인 처리 핸들러를 만들 필요없다.
        - 스프링 시큐리티는 알아서 들어오는 데이터로 처리한다.(로그아웃도 자동)
    * 그러나 로그인을 처리할 때 DB에 있는 정보를 참조해서 인증을 해야한다.
    * 따라서 DB에 있는 정보를 조회할 수 있는 UserDetailsService라는 것을 구현해야 한다.
    * 자동으로 bean으로 등록되고 UserDetailsService가 하나만 있으면 Spring Security설정을 해줄 필요 없다.
    * 로그인할 때 패스워드는 plain text지만 PasswordEncoder를 구현했으므로 굳이 명시적으로 설정해주지 않더라도 자동으로 encoding된다.
        - 참고) passwordEncoder가 여러 개거나 UserDetailsService가 여러 개라면 설정해야 한다.
```java
@Service
@RequiredArgsConstructor
public class AccountService implements UserDetailsService {

    ...

    @Override
    public UserDetails loadUserByUsername(String emailOrNickname) throws UsernameNotFoundException {
        Account account = accountRepository.findByEmail(emailOrNickname);
        if(account == null) { // 만약 이메일로 조회해서 null이면
            account = accountRepository.findByNickname(emailOrNickname); // 닉네임으로 조회
        }
        if(account == null) { // 닉네임으로 조회해도 null이면
            throw new UsernameNotFoundException(emailOrNickname); // emailOrNickname에 해당하는 유저가 없다는 예외 발생
        }
        return new UserAccount(account); // 유저가 있으면 pricipal에 해당하는 유저를 넘겨준다.
    }
}
```
- 실행하면 다음과 같이 로그인 페이지가 잘 만들어졌고 로그인, 로그아웃 기능도 정상적으로 동작한다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_14.jpg"></p>

<br>

## 로그인&로그아웃 테스트

### 목표
- 폼 서브밋 요청(post)은 반드시 `.with(csrf())`를 추가
    * csrf 토큰도 같이 전송한다.
- `.andExpect(authenticated())` 또는 `.andExpect(unauthenticated())`로 인증 여부 확인
- `.andExpect(status().is3xxRedirection())`로 리다이렉트 응답 확인
- `.andExpect(redirectedUrl())`로 리다이렉트 URL 확인
- JUnit 5의 `@BeforeEach`와 `@AfterEach` 사용
    * 사실 코드가 완전히 똑같은 테스트는 JUnit5가 제공하는 ParameterizedTest를 사용하는 것이 적절하다.
- 임의로 로그인 된 사용자가 필요한 경우에는 `@WithMockUser` 사용
<br>

### 구현
```java
@SpringBootTest
@AutoConfigureMockMvc
class MainControllerTest {

	@Autowired MockMvc mockMvc;
	@Autowired AccountService accountService;
	@Autowired AccountRepository accountRepository;

	@BeforeEach
	void beforeEach() {
		SignUpForm signUpForm = new SignUpForm();
		signUpForm.setNickname("khy07181");
		signUpForm.setEmail("khy07181@email.com");
		signUpForm.setPassword("12345678");
		accountService.processNewAccount(signUpForm);
	}

	@AfterEach
	void afterEach() {
		accountRepository.deleteAll(); // 테스트 마다 중복된 계정으로 signUpForm이 들어가므로 끝날 때마다 삭제
	}

	@DisplayName("이메일로 로그인 성공")
	@Test
	void login_with_email() throws Exception {
		mockMvc.perform(post("/login")
				.param("username", "khy07181@email.com")
				.param("password", "12345678")
				.with(csrf()))
				.andExpect(status().is3xxRedirection())
				.andExpect(redirectedUrl("/"))
				.andExpect(authenticated().withUsername("khy07181"));
	}

	@DisplayName("이메일로 로그인 성공")
	@Test
	void login_with_nickname() throws Exception {
		mockMvc.perform(post("/login")
				.param("username", "khy07181")
				.param("password", "12345678")
				.with(csrf()))
				.andExpect(status().is3xxRedirection())
				.andExpect(redirectedUrl("/"))
				.andExpect(authenticated().withUsername("khy07181"));
	}

	@DisplayName("로그인 실패")
	@Test
	void login_fail() throws Exception {
		mockMvc.perform(post("/login")
				.param("username", "111111")
				.param("password", "000000000")
				.with(csrf()))
				.andExpect(status().is3xxRedirection())
				.andExpect(redirectedUrl("/login?error"))
				.andExpect(unauthenticated());
	}

	@WithMockUser
	@DisplayName("로그아웃")
	@Test
	void logout() throws Exception {
		mockMvc.perform(post("/logout")
				.with(csrf()))
				.andExpect(status().is3xxRedirection())
				.andExpect(redirectedUrl("/"))
				.andExpect(unauthenticated());
	}
}
```
<br>

## 로그인 기억하기
- 세션이 만료 되더라도 로그인을 유지하고 싶을 때 사용하는 방법
    * 쿠키에 인증 정보를 남겨두고 세션이 만료 됐을 때에는 쿠키에 남아있는 정보로 인증한다.
- 1. 해시 기반의 쿠키
    * Username
    * Password
    * 만료 기간
    * Key
        - 애플리케이션 마다 다른 값을 줘야 한다.
        - 참고) 스프링 시큐리티 설정에서 `http.rememberMe().key("key 값")`으로 키 값만 설정하면 된다. -> 안전하지 않아서 쓸 일이 없을 듯하다.
    * 만약 쿠키를 탈취당하면? 계정을 뺏긴 것과 같다. 치명적인 단점이다.
- 2. 조금 더 안전한 방법은?
    * 쿠키안에 랜덤한 문자열(토큰)을 만들어 같이 저장하고 매번 인증할 때마다 바꾼다.
    * 쿠키 구성 : Username, 토큰
    * 그러나 이 방법도 취약하다. 
        - 쿠키를 탈취 당하면, 해커가 쿠키로 인증을 할 수 있고, 희생자는 쿠키로 인증하지 못한다.
- 3. 조금 더 개선한 방법
    * [Improved Persistent Login Cookie Best Practice](https://www.programering.com/a/MDO0MzMwATA.html)
    * 랜덤한 토큰 값을 쓰는 위의 방법에서 랜덤값이지만 추가로 고정된 시리즈라는 값을 사용한다.
        - 쿠키 구성 : Username, 토큰(랜덤, 매번 바뀜), 시리즈(랜덤, 고정)
        - 토큰은 매번 쿠키로 인증할 때마다 값이 바뀌고 시리즈는 바뀌지 않는다.
    * 쿠키를 탈취 당한 경우, **희생자는 유효하지 않은 토큰과 유효한 시리즈와 Username으로 접속하게 된다.**
    * 이 경우, 모든 토큰을 삭제하여 해커가 더이상 탈취한 쿠키를 사용하지 못하도록 방지할 수 있다.
        - 따라서 해커와 희생자 모두 쿠키로 로그인하지 못하고 희생자는 form기반의 로그인 창으로만 로그인할 수 있다.
- Spring Security는 해싱 기반의 쿠키 방법과 가장 안전한 방법 2개를 제공한다.
<br>

### 구현
- Spring Security 설정
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final AccountService accountService;
    private final DataSource dataSource;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        
        ...
        
        http.rememberMe()
                // tokenRepository를 쓸 때는 userDetailsService도 같이 설정해줘야 한다. 이미 accountService에 구현했으므로 주입받아 사용한다.
                .userDetailsService(accountService) 
                .tokenRepository(tokenRepository()); // username, 토큰, 시리즈 3개의 정보가 필요하다.
    }

    @Bean
    public PersistentTokenRepository tokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        jdbcTokenRepository.setDataSource(dataSource); // JDBC이므로 DataSource가 필요하고 JPA를 사용하기 때문에 기본적으로 bean으로 등록되어 있다.
        return jdbcTokenRepository;
    }

    ...
}
```
- JdbcTokenRepository가 사용하는 스키마가 있는데 이것에 맵핑이 되는 엔티티를 추가한다.
```java
@Table(name = "persistent_logins")
@Entity
@Getter @Setter
public class PersistentLogins {

    @Id
    @Column(length = 64)
    private String series;

    @Column(nullable = false, length = 64)
    private String username;

    @Column(nullable = false, length = 64)
    private String token;

    @Column(name = "last_used", nullable = false, length = 64)
    private LocalDateTime lastUsed;

}
```
- 로그인 뷰에 로그인 유지 체크 박스 추가
```html
<div class="form-group form-check">
    <input type="checkbox" class="form-check-input" id="rememberMe" name="remember-me" checked>
    <label class="form-check-label" for="rememberMe" aria-describedby="rememberMeHelp">로그인 유지</label>
</div>
```
- 실행하고 로그인 유지 체크박스를 체크하면 사용할 수 있고 세션Id를 삭제해도 로그인이 풀리지 않는다.
    * rememberMe가 있으므로 오히려 세션Id가 새로 생긴다.
<br>

## 프로필 뷰

### 목표
- 정보의 유/무 여부에 따라 보여줄 메세지가 다르다.
- 현재 유저가 프로필을 수정할 수 있는 권한이 있는지 판단해야 한다.
<br>

### 구현
- 프로필 뷰 맵핑 추가
    * 만약 nickname에 해당하는 account와 현재 조회를 하고 있는 @CurrentUser account가 일치하면 수정 권한이 있는 유저이다.
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    ...

    @GetMapping("/profile/{nickname}")
    public String viewProfile(@PathVariable String nickname, Model model, @CurrentUser Account account) {
        Account byNickname = accountRepository.findByNickname(nickname);
        if(nickname == null) {
            throw new IllegalArgumentException(nickname + "에 해당하는 사용자가 없습니다.");
        }

        model.addAttribute(byNickname);
        model.addAttribute("isOwner", byNickname.equals(account));
        return "account/profile";

    }
}
```
- 프로필 뷰 생성
    * account의 프로필 이미지가 비어있으면 기본 이미지로 설정
```html
<!DOCTYPE html>
<html lang="en"
      xmlns:th="http://www.thymeleaf.org">
<head th:replace="fragments.html :: head"></head>
<body class="bg-light">
<div th:replace="fragments.html :: main-nav"></div>
<div class="container">
    <div class="row mt-5 justify-content-center">
        <div class="col-2">
            <!-- Avatar -->
            <svg th:if="${#strings.isEmpty(account.profileImage)}" class="img-fluid float-left rounded img-thumbnail"
                 th:data-jdenticon-value="${account.nickname}" width="125" height="125"></svg>
            <img th:if="${!#strings.isEmpty(account.profileImage)}" class="img-fluid float-left rounded img-thumbnail"
                 th:src="${account.profileImage}"
                 width="125" height="125"/>
        </div>
        <div class="col-8">
            <h1 class="display-4 " th:text="${account.nickname}">Whiteship</h1>
            <p class="lead" th:if="${!#strings.isEmpty(account.bio)}" th:text="${account.bio}">bio</p>
            <p class="lead" th:if="${#strings.isEmpty(account.bio) && isOwner}">
                한 줄 소개를 추가하세요.
            </p>
        </div>
    </div>

    <div class="row mt-3 justify-content-center">
        <div class="col-2">
            <div class="nav flex-column nav-pills" id="v-pills-tab" role="tablist" aria-orientation="vertical">
                <a class="nav-link active" id="v-pills-intro-tab" data-toggle="pill" href="#v-pills-profile"
                   role="tab" aria-controls="v-pills-profile" aria-selected="true">소개</a>
                <a class="nav-link" id="v-pills-study-tab" data-toggle="pill" href="#v-pills-study"
                   role="tab" aria-controls="v-pills-study" aria-selected="false">스터디</a>
            </div>
        </div>
        <div class="col-8">
            <div class="tab-content" id="v-pills-tabContent">
                <div class="tab-pane fade show active" id="v-pills-profile" role="tabpanel" aria-labelledby="v-pills-home-tab">
                    <p th:if="${!#strings.isEmpty(account.url)}">
                            <span style="font-size: 20px;">
                                <i class="fa fa-link col-1"></i>
                            </span>
                        <span th:text="${account.url}" class="col-11"></span>
                    </p>
                    <p th:if="${!#strings.isEmpty(account.occupation)}">
                            <span style="font-size: 20px;">
                                <i class="fa fa-briefcase col-1"></i>
                            </span>
                        <span th:text="${account.occupation}" class="col-9"></span>
                    </p>
                    <p th:if="${!#strings.isEmpty(account.location)}">
                            <span style="font-size: 20px;">
                                <i class="fa fa-location-arrow col-1"></i>
                            </span>
                        <span th:text="${account.location}" class="col-9"></span>
                    </p>
                    <p th:if="${isOwner}">
                            <span style="font-size: 20px;">
                                <i class="fa fa-envelope-o col-1"></i>
                            </span>
                        <span th:text="${account.email}" class="col-9"></span>
                    </p>
                    <p th:if="${isOwner || account.emailVerified}">
                            <span style="font-size: 20px;">
                                <i class="fa fa-calendar-o col-1"></i>
                            </span>
                        <span th:if="${isOwner && !account.emailVerified}" class="col-9">
                                <a href="#" th:href="@{'/checkemail?email=' + ${account.email}}">가입을 완료하려면 이메일을 확인하세요.</a>
                            </span>
                        <span th:text="${#temporals.format(account.joinedAt, 'yyyy년 M월 가입')}" class="col-9"></span>
                    </p>
                    <div th:if="${isOwner}">
                        <a class="btn btn-outline-primary" href="#" th:href="@{/settings/profile}">프로필 수정</a>
                    </div>
                </div>
                <div class="tab-pane fade" id="v-pills-study" role="tabpanel" aria-labelledby="v-pills-profile-tab">
                    Study
                </div>
            </div>
        </div>
    </div>
</div>
</body>
</html>
```
- 실행 후 확인하면 이메일 인증을 해도 가입 날짜가 제대로 뜨지 않는다. (에러 발생)
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_15.jpg"></p>

<br>

## Open EntityManager (또는 Session) In View 필터
- JPA EntityManager(영속성 컨텍스트)를 요청을 처리하는 전체 프로세스에 바인딩 시켜주는 필터이다.
    * 기본적으로 스프링 부트 애플리케이션에서 활성화되어 있다.
    * 뷰를 랜더링 할때까지 영속성 컨텍스트를 유지하기 때문에 필요한 데이터를 랜더링 하는 시점에 추가로 읽어올 수 있다. (Lazy Loading)
    * 엔티티 객체 변경은 반드시 트랜잭션 안에서 해야 한다. 그래야 트랜잭션 종료 직전 또는 필요한 시점에 변경 사항을 DB에 반영한다.
- 현재의 에러는 컨트롤러에서 데이터를 변경했지만 DB에 반영이 되지 않았다.
    * **트랜잭션 범위 밖에서 일어난 일이기 때문**이다.
    * 영속성 컨텍스트가 관리하지 못한다.
- 따라서 스터디올래에서는 데이터 변경은 서비스 계층으로 위임해서 트랜잭션 안에서 처리한다.
    * 데이터 조회는 리파지토리 또는 서비스를 사용한다.
<br>

### 해결
- `completeSignUp()`과 `login()` 모두 accountService에 위임
```java
@Controller
@RequiredArgsConstructor
public class AccountController {

    ...

    @GetMapping("/check-email-token")
    public String checkEmailToken(String token, String email, Model model) {
        Account account = accountRepository.findByEmail(email);
        String view = "account/checked-email";
        
        if (account == null) {
            model.addAttribute("error", "wrong.email");
            return view;
        }
        if (!account.isValidToken(token)) {
            model.addAttribute("error", "wrong.token");
            return view;
        }

        accountService.completeSignUp(account); // accountService에서 모두 처리
        model.addAttribute("numberOfUser", accountRepository.count());
        model.addAttribute("nickname", account.getNickname());
        return view;
    }

    ...
}
```
- accountService에서 구현
    * 데이터를 변경하지 않고 조회만 하는 `loadUserByUsername()`에는 `@Transactional(readOnly = true)`를 붙인다.
```java
@Service
@Transactional // 트랜잭션 범위 안에서 관리
@RequiredArgsConstructor
public class AccountService implements UserDetailsService {

    ...

    public void completeSignUp(Account account) {
        account.completeSignUp();
        login(account);
    }
}
```
- 실행해서 확인하면 다음과 같이 가입 날짜가 잘 나오는 것을 확인할 수 있다.
<p align="center"><img src = "https://github.com/khy07181/TIL/blob/master/Spring_SpringBoot/img/signUp_16.jpg"></p>

<br>
