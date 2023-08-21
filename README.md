# 0818(Pooling기법, Cookie, Session, MVC, FrontController)

톰켓 홈 밑에 lib에 Driver를 넣어두었다.

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled.png)

그러면 서버에 모든 곳에서 DB로 접근이 가능함

DB 연결과 관련된 코드  = Business logic(DAO) 이라고 한다.

**Factory**는 미리 만들어둔 제품을 보관하는 개념

**Resource Factory**는 **Connection**들을 미리 가지고 있다.

Connection들은 이미 WAS에 등록을 해놔야 한다.

이것은 xml 파일을 사용한다.

Connection들을 찾아올 때는 `Context(인터페이스)` 클래스를 사용한다.

Context 메소드 중 `lookup()` 을 사용해 찾을 수 있다.

찾아서 반환 받을 때는 `DataSource` 타입이다. (**= Resource Factory**)

DataSource의 `getConnection()`을 사용하면 Connection을 얻을 수  있다.

이러한 방식을 **Pooling 기법**이라고 한다.

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
<style>
	#wrap{
		text-align:center;
		border: 2px dotted green;
	}
	h2{
		color:purple;
	}
</style>
</head>
<body>
<div id="wrap">
	<p><h2> DB연동으로 Cafe Member Manage</h2></p>
	<p><a href="register.jsp">회원 가입 하기</a></p>
	<p><a href="find.jsp">회원 검색 하기</a></p>
	<p><a href="AllMember">전체 회원 보기</a></p>
</div>
</body>
</html>
```

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%201.png)

다음과 같은 홈페이지에서 회원가입, 회원 검색하기, 전체 회원 보기 기능을 구현하려 한다.

먼저 위 기능을 구현할 DB 연결 코드를 만들어둔 DAO 클래스를 만들어야 한다.

```java
public class MemberDAOImpl implements MemberDAO{
	//필드 추가
	private DataSource ds;
	
	//싱글톤
	private static MemberDAOImpl dao = new MemberDAOImpl();
		//0. InitialContext 객체를 생성
	 	//1. DataSource를 하나 받아온다.
		
	private MemberDAOImpl() {
		//0. InitialContext 객체를 생성
	 	//1. DataSource를 하나 받아온다.
		try {
			InitialContext ic = new InitialContext();
			ds = (DataSource)ic.lookup("java:comp/env/jdbc/oracleDB"); //공장 찾음
			System.out.println("Datasource Lookup Sucess.....");

		}catch(NamingException e) {
			System.out.println("Datasource Lookup faill.....");
		}
	}
	
	public static MemberDAOImpl getInstance() { //싱글톤 
		return dao;
	}
	
	@Override
	public Connection getConnection() throws SQLException {		
		System.out.println("디비연결 성공....");
		return ds.getConnection(); //Connection 하나씩 Pool에서 받아온다..
	}

	@Override
	public void closeAll(PreparedStatement ps, Connection conn) throws SQLException{ //Factory에 Connection 반납 
		if(ps!=null) ps.close();		
		if(conn != null) conn.close();
	}

	@Override
	public void closeAll(ResultSet rs, PreparedStatement ps, Connection conn) throws SQLException{		
		if(rs != null)	rs.close();
		closeAll(ps, conn);		
	}
```

위 코드는 DB 연결을 위한 기본 코드이다. 싱글톤 기법으로 만들었다.

MemberDAOImpl의 생성자를 살펴보자.

`Context` 클래스를 사용해 `lookup()` 을 통해 Connection을 빌릴 Factory를 반환 받았다.

반환 받은 Factory는 필드값인 `DataSource ds` 에 넣었다.

그 다음 `getConnection()` 을 보면 Factory에 저장된 Connection 하나를 빌려온다.

### 회원가입

```java
@Override
	public void registerMember(MemberVO vo) throws SQLException { //회원 등록 
        Connection conn = null;
        PreparedStatement ps = null;
        try{
            conn = getConnection();
            String query = "INSERT INTO member (id, password, name, address) VALUES(?,?,?,?)";
            ps = conn.prepareStatement(query);

            ps.setString(1, vo.getId());
            ps.setString(2, vo.getPassword());
            ps.setString(3, vo.getName());
            ps.setString(4, vo.getAddress());

            System.out.println(ps.executeUpdate()+" row INSERT OK~~!!");
        }finally{
            closeAll(ps, conn);
        }
    }
```

이전에 JDBC에서 봤던 코드와 동일한다.

### 전체 회원 불러오기

```java
@Override
	public ArrayList<MemberVO> showAllMember() throws SQLException { //전체 멤버 조회 
		Connection conn = null;
		PreparedStatement ps = null;
		ResultSet rs = null;
		ArrayList<MemberVO> list = new ArrayList<>();
		try {
			conn = getConnection();
			String query = "SELECT id, password, name, address FROM member";
			ps = conn.prepareStatement(query);
			System.out.println("PreparedStatement....showAllMember()..");
			
			rs = ps.executeQuery();
			while(rs.next()) {
				list.add(new MemberVO(
						rs.getString("id"), 
						rs.getString("password"), 
						rs.getString("name"), 
						rs.getString("address")));
			}
		}finally {
			closeAll(rs, ps, conn);
		}
		return list;
	}
```

### ID 로 회원 찾기

```java
@Override
	public MemberVO findByIdMember(String id) throws SQLException { //id로 멤버 찾기 
		Connection conn = null;
		PreparedStatement ps = null;
		ResultSet rs = null;
		MemberVO vo = null;
		try{
			conn=getConnection();
			String query = "SELECT id, password, name, address FROM member WHERE id=?";
			ps = conn.prepareStatement(query);
			
			ps.setString(1,  id);
			rs = ps.executeQuery();
			if(rs.next()) {
				vo=new MemberVO(id,
								rs.getString("password"),
								rs.getString("name"),
								rs.getString("address"));
			}
			System.out.println(id + ", findByIdMember Sucess");
			
		}finally{
			closeAll(rs, ps, conn);
		}
		return vo;
	}
```

이제 Business logic을 살펴보았으므로 Container를 살펴본다.

먼저 위에 메인 화면에서 ‘회원가입하기’ 버튼을 누르게 되면 `register.jsp` 로 이동하게 된다.

```html
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
<style type="text/css">
	h2{
		text-align: center;
		color: purple;
	}
	#wrap{
		margin-left: 220px;		
	}
</style>
<script type="text/javascript">
	function btnclick(){
		alert("button Click~~!!!");
	}
</script>
</head>
<body>
	<h2>REGISTER MEMBER FORM</h2>
	<div id="wrap">
		<form action="Register" method="post">
			ID <input type="text" name="id" required="required"><br><br>
			PASS <input type="password" name="password" required="required"><br><br>
			NAME <input type="text" name="name" required="required"><br><br>
			ADDR <input type="text" name="address" required="required"><br><br>
			<input type="submit" value="REGISTER">
			<input type="button" value="CLICK" onclick="btnclick()">
		</form>
	</div>
</body>
</html>
```

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%202.png)

ID, PASS, NAME, ADDR를 누르고 REGISTER 버튼을 누르면 /Register 주소로 이동하게 된다. 

그럼 해당 form 값을 받고 데이터 처리를 할 Servlet을 만들어 준다.

```java
@WebServlet("/Register")
public class RegisterServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;
       
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doProcess(request, response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      doProcess(request, response);
	}

	protected void doProcess(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
      request.setCharacterEncoding("utf-8");
      response.setContentType("text/html;charset=utf-8");
      //로직은 여기에 작성
      
      //1. 폼값 받아서 
      String id = request.getParameter("id");
      String password = request.getParameter("password");
      String name = request.getParameter("name");
      String address = request.getParameter("address");

      //2. VO 생성... PVO
      MemberVO pvo = new MemberVO(id, password, name, address);
      String path = "index.html";
      //3. DAO 리턴받고 비즈니스 로직 호출
      try {
    	  MemberDAOImpl.getInstance().registerMember(pvo);
    	  //path = "register_result.jsp"; //굳이 결과 페이지 필요없음
    	  //path="allView.jsp"; 이렇게 하면 500 오류가 난다. allView.jsp는 전체 멤버를 조회하는 DAO를 사용해야 한다. 따라서 AllMemberServlet을 갔다가 allView.jsp로 가야한다.
      }catch(Exception e) {
    	  
      }
      //4. 바인딩?? => 필요없다. 
      
      //5. 네비게이션 register_result.jsp (굳이 결과페이지 필요없음)
      //request.getRequestDispatcher(path).forward(request, response); 
      response.sendRedirect("AllMember");
	}
}
```

form값을 받고 비즈니스 로직인 `registerMember()` 를 호출해 회원등록을 한다.

그 다음 회원가입이 완료했다는 성공 페이지를 보여줘야 하는데 굳이 필요없는 기능이기 때문에

자신이 회원가입이 됐다는 것을 보여주기 위해 전체 회원을 보여주는 편이 낫다.

따라서 `request.getRequestDispatcher("AllMember").forward(request, response)` 를 호출하는것이 아니라 `response.sendRedirece("AllMember")` 를 호출해야 한다.

Client로 다시 가서 **AllMember** Servlet을 거쳐서 그 안에서 `showAllMember()` 로직을 호출해야 

전체 회원을 조회할 수 있기 때문이다.

메인화면에서 ‘전체 회원 조회’ 를 선택하면 AllMember 주소로 이동한다.

그럼 AllMember Servlet을 살펴본다.

```java
@WebServlet("/AllMember")
public class AllMemberServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doProcess(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");
		
		//1. DAO 리턴 받고 business logic 호출
		//2. 반환된 값 바인딩
		//3. 결과 페이지로 네비게이션 ...allView.jsp
		
		try {
			ArrayList<MemberVO>list =MemberDAOImpl.getInstance().showAllMember();
			request.setAttribute("list", list);
			request.getRequestDispatcher("allView.jsp").forward(request, response);
		}
		catch(Exception e) {
			
		}
	}
}
```

ArrayList를 이용해 `showAllMember()` 의 return값을 받는다.

DB를 통해 받아온값은 `request` 객체에 넣을 수 없으므로 `setAttribute()` 를 통해 넣어준다.

그래야 allView.jsp 에서 해당 값을 가져올 수 있기 때문이다.

```html
<%@page import="servlet.model.MemberVO"%>
<%@page import="java.util.ArrayList"%>
<%@ page language="java" contentType="text/html; charset=UTF-8"
	pageEncoding="UTF-8"%>
<%
ArrayList<MemberVO> list = (ArrayList) request.getAttribute("list");
%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
<meta name="viewport" content="width=device-width, initial-scale=1">

<link rel="stylesheet"
	href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.2/dist/css/bootstrap.min.css">
<script
	src="https://cdn.jsdelivr.net/npm/jquery@3.6.4/dist/jquery.slim.min.js"></script>
<script
	src="https://cdn.jsdelivr.net/npm/popper.js@1.16.1/dist/umd/popper.min.js"></script>
<script
	src="https://cdn.jsdelivr.net/npm/bootstrap@4.6.2/dist/js/bootstrap.bundle.min.js"></script>
</head>
<body>
	<!--나중에 이부분은 BootStrap 클래스 속성 연결해서 완전한 디자인으로 직접 만들어 주세요  -->
	<div class="jumbotron text-center">
		<h2>회원 전체 명단 보기</h2>
	</div>
	<div class="container">
	<table class="table table-hover">
		<thead>
			<tr>
				<th>ID</th>
				<th>이름</th>
				<th>주소</th>
			</tr>
		</thead>
		<tbody>
			<%
			for (MemberVO vo : list) {
			%>
			<tr>
				<td><%=vo.getId()%></td>
				<td><%=vo.getName()%></td>
				<td><%=vo.getAddress()%></td>
			</tr>
			<%
			}
			%>
		</tbody>
	</table>
	</div>
</body>
</html>
```

BootStrap을 활용해 테이블로 출력해주었다.

이제 메인화면에서 ‘회원 검색 하기’ 버튼을 누르면 find.jsp로 이동한다.

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%203.png)

ID를 작성하고 제출 버튼을 누르면 /Find로 이동한다.

```html
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
<h2>단순 회원 검색하기</h2>
<form action="Find" method="post">
	조회할 ID <input type="text" name="id" required="required">
	<input type="submit" name="회원조회">
</form>
</body>
</html>
```

Find Servlet을 살펴보자.

```java
@WebServlet("/Find")
public class FindServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;
       
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		doProcess(request, response);
	}

	
	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		doProcess(request, response);

	}
	protected void doProcess(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
        response.setContentType("text/html;charset=utf-8");
        
        //로직은 여기서 작성
        //1 getParameter로 ID 받아오기 -Front와 연결
        String id = request.getParameter("id");
        
        //2 DB 에 해당 ID 있는지 확인 -- DB 연결
        String path="find_fail.jsp";
        try {
        	MemberVO rvo = MemberDAOImpl.getInstance().findByIdMember(id);
        	if(rvo != null) { //ID를 통해 회원을 찾으면
        		request.setAttribute("vo", rvo);//3 반환된 값을 바인딩
        		path="find_ok.jsp";
        	}
        }
        catch(Exception e) {
        	
        }        
        //4 네이게이션...jsp 결과페이지로 -- View와 연결 
		request.getRequestDispatcher(path).forward(request, response);
	}
}
```

`findByIdMember()` 를 활용해 ID를 통해 회원을 찾는다.

rvo가 null 이 아니면 회원을 찾았다는 의미이므로 rvo를 `setAttribute()` 로 저장해주고

path를 find_ok.jsp로 설정한다.

```html
<%@page import="servlet.model.MemberVO"%>
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
	<%
		MemberVO vo =(MemberVO)request.getAttribute("vo");
	%>
	<h2>회원 검색 결과</h2>
	ID : <%= vo.getId() %><br>
	NAME :<%= vo.getName() %><br>
	ADDRESS: <%= vo.getAddress() %> 
</body>
</html>
```

# Session Management With Cookie

Attribute 에는 데이터 유효기간에 따라 크게 3가지로 나뉜다.

1. **ServletRequest**
    
    응답 전 까지 데이터 보관
    
2. **HttpSession**
    
    로그인 하는 동안 데이터 보관
    
    ?? 로그인 상태라는 것을 어떻게 알까?
    
    이전의 요청한 사용자의 정보와 방금 요청한 사용자의 정보가 같다는 것을 인식하는것이
    
    로그인이다.
    
3. **ServletContext**
    
    서버가 멈추기 전까지 정보보관
    

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%204.png)

### Cookie

1. 쿠키는 A Server에서 만들어 진다.  정보가 String으로 저장된다.
    
    (A 서버의 정보가 쿠키에 들어감)
    
    `Cookie c = new Cookie(”id”, “kb”);`
    
2. Server가 응답하면 쿠키는 브라우저로 보내지게 된다.
    
    `response.addCookie(c);`
    
3. B server로 요청때 브라우저에 저장된 쿠키가  B Server로 전달된다.
    
    전달 될 때 모든 쿠키가 간다. 그중에서 내가 원하는 쿠키를 찾는다.
    
    `Cookie[] cookies = request.getCookies();` // 브라우저에 저장된 모든 쿠키
    
    결국 A서버에서 만든 정보가 B 서버로 데이터 전달이 일어난다.
    

그렇다면 Attribute에서 데이터 전달과 Cookie에서 데이터 전달의 차이는 무엇인가?

찾아보기

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%205.png)

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
</head>
<body>
<h2>Cookie API</h2>
<a href="CookieServlet">Cookie Creating and sent by a servlet to a Web browser</a>
</body>
</html>
```

버튼을 누르면 CookieServlet으로 이동한다.

```java
@WebServlet("/CookieServlet")
public class CookieServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;
       
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
        response.setContentType("text/html;charset=utf-8");
        
        //1. 쿠키 생성
        Cookie c1 = new Cookie("id", "KBLife");
        Cookie c2 = new Cookie("today", "2023-08-18");
       
        //쿠키안에 저장된 정보를 유지하는 기간을 지정
        c1.setMaxAge(24*60*60); //하루동안 정보 보관
        c2.setMaxAge(2*24*60*60); //2일 동안 정보 보관
        
        //2. 생성된 쿠키를 클라이언트로 보냄 ... 브라우저에 저장
        response.addCookie(c1);
        response.addCookie(c2);
        
        //3, 페이지 이동... redirect로 해야함 쿠키를 결과페이지에 전달해야 하기 때문 
        response.sendRedirect("getCookie.jsp");
        
	}

}
```

현재 쿠키는 브라우저에 있으므로 결과 페이지로 바로 forward 방식이 불가능한다.

따라서 Redirect를 사용해 브라우저로 갔다가 getCookie.jsp로 가야한다.

```html
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%
Cookie[]cs = request.getCookies();
for(Cookie c : cs){
%>
	<li>Name : <%= c.getName() %></li>
	<li>Value : <%= c.getValue() %></li>
	
<%
}
%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>

</body>
</html>
```

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%206.png)

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%207.png)

### Session

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%208.png)

브라우저가 Server로 요청을 하면 Request, Response, thread, Session이 만들어진디ㅏ.

이 때 Session에 값이 자동으로 들어간다.

이게 바로 JSESIONID 값이다.

…

…

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%209.png)

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
<h2>Login Page</h2>
<form action="LoginServlet" method="post">
	ID : <input type="text" name="id" required="required"><br><br>
	PASSWORD : <input type="password" name="password" required="required"><br><br>
	<input type="submit" value="Login">
</form>
</body>
</html>
```

여기서 form값을 입력하고 Login 버튼을 누르면 LoginServlet으로 이동한다.

```java
@WebServlet("/LoginServlet")
public class LoginServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;
       
	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		 request.setCharacterEncoding("utf-8");
	     response.setContentType("text/html;charset=utf-8");
	     
	     /*
	      1. 폼값 받아서...
	      2. DAO 리턴받고.. 비즈니스 로직 호출...
	      3. 반환값 바인딩
	      4. 결과 페이지로 네비게이션
	      */
	     
	     //세션은 클라이언트가 서버에 요청시에 서버에 만들어진다.
	     //만들어진 세션을 받아서 사용한다.
			//login.html에서 LoginServlet으로 요청해서 세션 생성됨
	     HttpSession session =request.getSession(); 
	     
	     System.out.println("JSESSION::" + session.getId()); //JSESSION 확인
	     
	     String id = request.getParameter("id");
	     String password = request.getParameter("password");
	     
	     MemberVO vo = new MemberVO(id, password, "길복순", "여의도");
	     
	     //비즈니스 로직 호출... 결과값 반환...
	     
	     //바인딩 ******************!!
	     //attribute를 session에 바인딩 하는 경우 2가지 
	     // 로그인, 회원정보 수정 이 2개 말고 나머지는 경우 없다.
	     session.setAttribute("vo", vo); //세션에 현재 사용자 정보 저장
	     
	     //네비게이션
	     response.sendRedirect("BuyServlet");
	}
}
```

현재 사용자가 로그인한 사용자인지 확인하기 위해 Client를 갔다가 다시 책을 사기 위해 BuyServlet으로 

이동한다.

그럼 BuyServlet에서 현재 Session을 확인해 이전에 로그인 한 회원인지 판별해야 한다.

```java
@WebServlet("/BuyServlet")
public class BuyServlet extends HttpServlet {
	protected void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doProcess(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");
		
		//로직은 여기서 작성... 이것은 새로운 세션이 아니라 이전 세션일 것이다...확인 하자
		HttpSession session = request.getSession();
		
		if(session.getAttribute("vo") != null) { //로그인 된 상태라면
			System.out.println("JSESSIONID... ButServlet" + session.getId());
			session.setAttribute("book", "오펜하이머");
			request.getRequestDispatcher("buy_result.jsp").forward(request, response);

		}
		else { //로그인 안된 상태라면... 다시 로그인 하러 보내야 함
			response.sendRedirect("login.html");
		}
	}
}
```

BuyServlet에서 뽑아낸 Session에서 `getAttribute()` 를 했을 때 null 이 아니면 이전에 로그인한

회원이라는 의미이다.

여기서 현재 회원이 “오펜하이머” 라는 책을 샀다고 `setAttribute()` 해주고 buy_result.jsp로 forward 한다

```html
///buy_result.jsp
<%@page import="servlet.model.MemberVO"%>
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<%
	MemberVO vo=(MemberVO)session.getAttribute("vo");
	String book=(String)session.getAttribute("book");
	if(vo==null){ //로그인 한 상태가 아니라면
%>
	<h3>로그인부터 하세여</h3>
	<a href="login.html">LOGIN</a>
<%
	}
%>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR"> 
<title>Insert title here</title>
</head>
<body>
<h2>Information...</h2>
LOGIN ID : <b><%= vo.getId() %></b><br>
LOGIN Name : <b><%= vo.getName() %></b><br>
ProductName : <b><%= book %></b><br>
</body>
</html>
```

위에 메인 페이지에서 ID는 “강유석”으로 입력하고 PASSWORD는 1234라고 입력하고

Login 버튼을 누르게 되면

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2010.png)

위 사진처럼 나오고 console을 확인해 보면

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2011.png)

LoginServlet의 세션과 BuyServlet의 세션이 같다는 것을 알 수 있다.

# MVC

지금까지 코드를 MVC 패턴으로..

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2012.png)

## Model(MemberVO, MemberDAO, MemberDAOImpl)

위에서 본 MemberDAOImpl 에서 로그인 기능이 추가되었다.

### 로그인

```java
@Override
	public MemberVO login(String id, String password) throws SQLException {
		Connection conn = null;
		PreparedStatement ps = null;
		ResultSet rs = null;
		MemberVO vo = null;
		try{
			conn=getConnection();
			String query = "SELECT id, password, name, address FROM member WHERE id=? and password=?";
			ps= conn.prepareStatement(query);
			ps.setString(1, id);
			ps.setString(2, password);
			
			rs=ps.executeQuery();
			if(rs.next()) {
				vo=new MemberVO(id,
								password,
								rs.getString("name"),
								rs.getString("password"));
						
			}
		}finally {
			closeAll(rs, ps, conn);
		}
		return vo;
	}
```

## Controller(Servlet)

마찬가지로 controller에서 LoginServlet이 추가 되었다.

```java
@WebServlet("/LoginServlet")
public class LoginServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doProcess(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");
		
		//로직은 여기서 작성...
		/*
		 1. 폼값 받아서 
		 2. DAO 리턴받고 비즈니스 로직 호출
		 3. 반환값 받아서 ... null이 아니면 session에 바인딩... 결과페이지(login_result.jsp)로 네비게이션
		 4. null 이 아니면 다시 로그인 페이지(login.jsp)로 이동
		 */
		String id=request.getParameter("id");
		String password=request.getParameter("password");
		String path="index.html";
		
		try {
			MemberVO rvo = MemberDAOImpl.getInstance().login(id,  password);
			HttpSession session = request.getSession();
			
			if(rvo!=null) {
				session.setAttribute("vo", rvo);
				System.out.println("JSESSIONID ::" + session.getId());
				path="login_result.jsp";
			}
		}catch(Exception e) {
			path="login.jsp";
		}
		request.getRequestDispatcher(path).forward(request, response);
	}
}
```

## View (.jsp)

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2013.png)

저기에서 로그인 하기 버튼을 누르면

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2014.png)

```html
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
 <meta name="viewport" content="width=device-width, initial-scale=1">
 <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.2/dist/css/bootstrap.min.css">
 <script src="https://cdn.jsdelivr.net/npm/jquery@3.6.4/dist/jquery.slim.min.js"></script>
 <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.1/dist/umd/popper.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.6.2/dist/js/bootstrap.bundle.min.js"></script>
  <style type="text/css">
  .container{
  	width: 600px;
  	margin: 0 auto;
  }
  </style>
<title>Insert title here</title>
</head>
<body>
<div class="container">
	<h2 align="center">Login Page</h2>
	<form action="LoginServlet" method="post">
	<div class="form-group">
      <label for="ID">ID:</label>
      <input type="text" class="form-control" id="id" placeholder="Enter id" name="id">
    </div>
    <div class="form-group">
      <label for="pwd">Password:</label>
      <input type="password" class="form-control" id="password" placeholder="Enter password" name="password">
    </div>
    <div class="form-group form-check">
      <label class="form-check-label">
        <input class="form-check-input" type="checkbox" name="remember"> Remember me
      </label>
    </div>
    <button type="submit" class="btn btn-primary">LOGIN</button>
	</form>
</div>
</body>
</html>
```

로그인 폼을 작성하고 Login 버튼을 누르면 LoginServlet으로 이동한다.

그곳에서 정상적으로 로그인되면 login_result.jsp로 forward 된다.

```html
<%@page import="servlet.model.MemberVO"%>
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<%
	MemberVO vo=(MemberVO)session.getAttribute("vo"); 
%>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
</head>
<body>
<h2>Login Information</h2>
ID <%= vo.getId() %><br>
NAME <%=  vo.getName() %><br>
ADDRESS <%= vo.getAddress() %><br>
<p></p><hr><p></p>
<a href="logout.jsp">LOG OUT</a>
<a href="index.html">INDEX</a>
</body>
</html>
```

여기서 로그아웃 버튼을 누르면 logout.jsp로 이동한다.

```html
<%@page import="servlet.model.MemberVO"%>
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<%
	MemberVO vo=(MemberVO)session.getAttribute("vo"); 
	if(vo==null){ //로그인 안된상태라면...로그인 하기로
%>
	<h2><a href="login.jsp">로그인 하기</a></h2>
<%
	}else{  //로그인 된 상태라면....로그아웃 진행(세션을 Death)
		session.invalidate();//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
	}
%>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
<script type="text/javascript">
	function logout() {
		alert("Log Out~~")
	}
</script>
</head>
<body onload="return logout()">
<b>로그아웃 되셨습니다...</b><br>
<a href="index.html">INDEX</a>
</body>
</html>
```

로그아웃을 하려면 현재 Session을 제거해야 한다. `session.invalidate()` 를 통해

현재 Session을 삭제할 수 있다.

# FrontController

![Untitled](0818(Pooling%E1%84%80%E1%85%B5%E1%84%87%E1%85%A5%E1%86%B8,%20Cookie,%20Session,%20MVC,%20FrontCont%206b0a343ad69d4ecea84c7d3cb8d5085e/Untitled%2015.png)

위의 코드를 보면 Servlet이 총 4개 나오게 된다.

하나의 Servlet을 만들면 여러 인터페이스나, 요청을 받게 되면 Request, Response, thread, Session이 만들어진다. 

이게  x4가 만들어 지는 것이다.

즉, Servlet이 너무 만들어지는 문제점이 존재한다.

따라서 4개의 Servlet을 1개의 Servlet으로 묶을 것이다.

그렇게 되면 Servlet은 어떤 요청이 오는지 하나하나 판별을 해줘야 한다.

⇒ FrontController 패턴으로 만들어주면 된다.

Servlet으로 요청을 모든 곳에서 FrontController로 가게 하는 것이다.

find.jsp, login.jsp, register.jsp, allView.jsp 에서 요청을 각각의 Servlet이 아니라

[front.do](http://frond.do) 라는 FrontCotroller로 가는 것이다.

그러면 먼저 어떤 요청이 왔는지 알아야 한다.

find.jsp에는

```html
<input type="hidden" name="command" value="find">
```

해당 코드를 form 태그 안에 넣어 준다. 위 코드는 hidden 이기 때문에 Client 화면에는 보이지 않는다.

login.jsp에는

```html
<input type="hidden" name="command" value="login">
```

register.jsp에는

```html
<input type="hidden" name="command" value="register">
```

전체 회원 조회는 index.html에서  `?` 를 이용해 작성한다.

```html
<p><a href="front.do?command=showAll">전체 회원 보기</a></p>
```

이제 어떤 요청이 알았으므로 요청에 따른 Servlet을 FrontController에 정의해주면 된다.

```java
@WebServlet("/front.do")
public class FrontController extends HttpServlet {
	protected void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		doProcess(request, response);
	}

	protected void doProcess(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");

		// 로직은 여기서 작성...어떤 요청이 들어왔는지를 ... 구분
		// register, login, find, showAll...
		String command = request.getParameter("command");

		String path = "index.html";
		if (command.equals("register")) { // 회원가입 로직..
			path = register(request, response);
		} else if (command.equals("find")) {
			path = find(request, response);
		} else if (command.equals("login")) {
			path = login(request, response);
		} else if (command.equals("showAll")) {
			path = showAll(request, response);
		}
		request.getRequestDispatcher(path).forward(request, response);

	}// do process

	private String register(HttpServletRequest request, HttpServletResponse response)
			throws IOException, ServletException {
		// 1. 폼값 받아서
		String id = request.getParameter("id");
		String password = request.getParameter("password");
		String name = request.getParameter("name");
		String address = request.getParameter("address");

		// 2. VO 생성... PVO
		MemberVO pvo = new MemberVO(id, password, name, address);
		String path = "index.html";
		// 3. DAO 리턴받고 비즈니스 로직 호출
		try {
			MemberDAOImpl.getInstance().registerMember(pvo);
			
		} catch (Exception e) {

		}
		return path;
	}

	private String find(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		// 로직은 여기서 작성
		// 1 getParameter로 ID 받아오기 -Front와 연결
		String id = request.getParameter("id");

		// 2 DB 에 해당 ID 있는지 확인 -- DB 연결
		String path = "find_fail.jsp";
		try {
			MemberVO rvo = MemberDAOImpl.getInstance().findByIdMember(id);
			if (rvo != null) {
				request.setAttribute("vo", rvo);// 3 반환된 값을 바인딩
				path = "find_ok.jsp";
			}
		} catch (Exception e) {

		}
		// 4 네이게이션...jsp 결과페이지로 -- View와 연결
		return path;
	}

	private String login(HttpServletRequest request, HttpServletResponse response) {
		String id = request.getParameter("id");
		String password = request.getParameter("password");
		String path = "index.html";

		try {
			MemberVO rvo = MemberDAOImpl.getInstance().login(id, password);
			HttpSession session = request.getSession();

			if (rvo != null) {
				session.setAttribute("vo", rvo);
				System.out.println("JSESSIONID ::" + session.getId());
				path = "login_result.jsp";
			}
		} catch (Exception e) {
			path = "login.jsp";
		}

		return path;
	}

	private String showAll(HttpServletRequest request, HttpServletResponse response) {
		String path="index.html";
		try {
			ArrayList<MemberVO> list = MemberDAOImpl.getInstance().showAllMember();
			request.setAttribute("list", list);
			path = "allView.jsp";
		} catch (Exception e) {

		}
		return path;
	}
}
```

이렇게 작성하면 결국 Servlet은 FrontController 1개만 만든것과 같다.
