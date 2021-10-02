---
title: "Linux 유저 관리"
date: 2021-10-02 22:12:00 +0900
categories: linux
---

 이전 회사나, 현재 회사 모두 사내에 워크스테이션을 (linux server) 가지고 있고, 팀원 분들에게 각각 계정이 발급되어 사용하는 구조이다.
대부분의 경우, 데이터셋과 같은 공유 자료 접근을 위해 하나의 서버에 NFS Server 를 구성해두고, 다른 서버에서 이를 mount 해서 사용하는 구조이다.

 흔히 겪는 문제로는, NFS 서버 사이에 uid, gid 가 꼬여서 분명 (다른 서버의) 내가 작성한 파일이라도 `Permission Denied` 로 접근이 불가능한 경우이다. `sudo` 로 해결이 가능하지만, 당연히 보안상 모두에게 `sudoer` 를 주는 것은 좋지 못할 뿐만 아니라, 사용자 입장에서도 매번 번거롭기 때문에 이같은 문제가 발생하지 않도록 세팅하는게 팀의 생산성을 높이는데 큰 도움이 된다.

## UID? GID?

 UNIX 에서는 유저마다 할당된 UID, GID 로 권한 등을 관리한다. 다음은 mac 에서 `ls -alh` 를 수행한 결과이다.

![mac-ll.png](/assets/images/2021-10-02-linux-users-management/mac-ll.png)

이에 대해 설명하자면, 맨 앞의 컬럼은 권한 정보를 나타낸다. d는 폴더라는 것을 뜻하고, (여기서 -는 파일), 그 뒤의 rwx 는 각각 Read/Write/eXecute 를 나타낸다.
 흔히 `chmod +x ~`, `chmod 755 ~` 등은 이를 나타내는데, 3개 위치의 rwx 각각 3비트라고 생각해서 읽은 숫자이다. (ex. rwx => 7, r-x => 5)

각 rwx 들은 순서에 따라 해당 폴더/파일의 Owner/Group/Others 에 매핑된다. `daangn` 이라는 그룹에 속한 `jeremy` 라는 유저가 만든 `-rwxr-xr--` 인 파일이 있다고 하자. 그러면, `jeremy` 본인은 Read/Write/eXecute 모두 가능하고, 그 외 `daangn` 에 속한 유저는 Read/eXecute, 나머지 유저는 Read 만 가능하다.

## NFS 서버에서 문제점

 그렇다면 왜 위에서 언급한 NFS Server 에서의 문제점이 발생하는 걸까?

NFS mount 된 개별 서버의 User, Group 은 각각의 서버에서 개별적으로 관리하는 것이기 때문이다. 우리 눈에는 `jeremy` 라는 유저로 보이지만, 다음과 같이 실제로는 `uid=501, gid=20` 으로 관리된다.

```bash
➜  seo-jinBro.github.io git:(main) ✗ id
uid=501(jeremy) gid=20(staff) groups=20(staff)
```

 즉, 각 서버의 `jeremy` 들은 모두 같은 `jeremy` 가 아니고, A 서버에서 `jeremy` 는 `uid=1001`, B 서버에서 `jeremy` 는 `uid=1007` 과 같은 식인 것이다. 이 경우, A 서버에서 `jeremy` 가 파일을 쓰는 경우 A 서버에서는 `jeremy` 로 잘 보이고, `chmod` 로 설정한 권한이 잘 동작하지만 B 서버에서는 이 파일의 소유자는 `1001번 유저` 이므로, `jeremy` 는 해당 파일의 소유자가 아니다. 따라서 해당 파일에 소유자 권한이 없다.


## How to?

 따라서, 모든 서버의 uid, gid 를 유저마다 통일하는 것이 중요하다. 이를 관리하기 위해서 다음과 같은 형태로 사용 중인데, 현재까지 꽤나 괜찮은 것 같다.

아래는 중앙화해서 관리하는 `users.txt` 이다.

```bash
# 공통
<common> 2000

# ml팀
<ml_team_0> 2001
<ml_team_0> 2002
...

# not-ml
<data_team_0> 3000
<data_team_1> 3001
...

# 인턴 팀원분들 (자주 삭제가 될 수 있으므로 따로 분류)
<intern_0> 4000
...
```

다음은 세팅할 수 있는 bash script `settings.sh` 이다.

```bash
#!/bin/bash
set -e

while read line; do
    if [[ $line =~ ^\#.* || -z $line ]]; then
        continue
    fi
    arr=($line)
    username=${arr[0]}
    uid=${arr[1]}

    if id -u "$username" >/dev/null 2>&1; then
        continue
    fi

    if [[ $username == "<common>" ]]; then # 위의 common 에 해당
        useradd $username -p $username -M -s /bin/bash -G docker -U -u $uid
        groupmod -g $uid $username
        echo "Add new user: $username, uid: $uid"
    else
        useradd $username -N -p $(openssl passwd -crypt $username) -m -s /bin/bash -g <group> -G docker -u $uid # 관리하는 group 명
        echo "Add new user: $username, uid: $uid"
    fi
done <$1
```

위 두 파일을 `/` 에 넣어두거나, ssh 로 전송해서 각 서버에서 실행시킬 수 있다. 전자의 경우라면 다음과 같다.

```bash
$ sudo su; /settings.sh /users.txt
```

이후 새로 추가된 유저분들에게는 초기 비밀번호가 `$username` 이라는 것을 안내드리고, 접속 확인 후 `passwd` 로 비밀번호를 바꿔달라고 요청드린다. (Optional) `ssh-copy-id $username@$server` 로 ssh 세팅도.

## 기존 유저 UID, GID 변경

 현재 이미 유저가 세팅되어있고, 꼬여있다면 이를 해결해야하는데, 편의상 유저 이름만 있는 test-users.txt 라는 파일이 있다 가정하면 다음과 같이 할 수 있다. (**주의: 상황에 따라 다를 수 있으므로, 신중히 하는 것을 권고.**)

```bash
$ cat only-name-users.txt | xargs -I {} sudo usermod -g <group> {}; # 원하는 group 명
$ cat only-name-users.txt | xargs -I {} sudo groupdel {};
```
