> [신입 프로그래머를 위한 실전 JSP 강좌 강의](https://www.inflearn.com/course/%EC%8B%A4%EC%A0%84-jsp-%EA%B0%95%EC%A2%8C/dashboard)를 듣고 공부한 내용을 정리하여 기록

<br>

### url-pattern

##### 디렉터리 패턴
- 디렉터리 형태로 서버의 해당 컴포넌트(Servlet)를 찾아서 실행하는 구조
- 각 디렉토리의 맵핑명으로 다른 서블릿을 찾아간다.
<br>

##### 확장자 패턴
- 확장자 형태로 서버의 해당 컴포넌트(Servlet)를 찾아서 실행하는 구조
- 확장자만 같으면 확장자 패턴으로 Servlet 맵핑이 되어있는 서블릿으로 찾아가 그 안에서 분기한다.
- MVC 모델에서 많이 쓰인다.
<br>

### FrontController 패턴
- 클라이언트의 다양한 요청을 한 곳으로 집중시켜, 개발 및 유지보수에 효율성을 극대화하는 방법
![](https://github.com/khy07181/TIL/blob/master/Servlet_JSP/img/FrontControllerPattern_CommandPattern_1.png)
<br>

### Command 패턴
- 클라이언트로부터 받은 요청들에 대해서, 서블릿이 작업을 직접 처리하지 않고, 해당 클래스가 처리하도록 하는 방법
![](https://github.com/khy07181/TIL/blob/master/Servlet_JSP/img/FrontControllerPattern_CommandPattern_2.png)
