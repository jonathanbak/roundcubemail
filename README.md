# roundcubemail
웹메일운영 (Sendmail+dovecot+roundcubemail)

sendmail+dovecot으로 smtp, imap을 이용하여 아웃룩등에 설정하여 사용하다가 컴알못 회사 동료들에게 시달리기 싫어 웹메일을 열어줘 봅니다.

구글링하면 잘 정리된 글들이 많지만 roundcubemail에 대해서는 한글로 정리된 것이 없는것 같아 한번 정리해봅니다.

CentOS 6.8
sendmail-8.14.4
dovecot-2.0.9
roundcubemail 1.2.4-complete

##### Sendmail 설치 (SMTP)

    # yum install make sendmail sendmail-cf

###### 설정
    # cd /etc/mail
    # mv sendmail.cf sendmail.cf.org  //백업
    # vi sendmail.mc
**sendmail.mc* 파일에서 `dnl` 은 주석을 뜻합니다

    (Before) dnl TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
    (Before) dnl define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
    (After) TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
    (After) define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl

    (Before) DAEMON_OPTIONS(`Port=smtp,Addr=127.0.0.1, Name=MTA')dnl
    (After) DAEMON_OPTIONS(`Port=smtp,Addr=0.0.0.0, Name=MTA')dnl

    (Before) dnl MASQUERADE_AS(`mydomain.com')dnl
    (After) MASQUERADE_AS(`devd.kr')dnl

    (Before) dnl FEATURE(masquerade_envelope)dnl
    (After) FEATURE(masquerade_envelope)dnl

###### 설정 컴파일 적용
    # make
또는

    # m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf

###### 메일 송신 허용 호스트 설정
    # vi /etc/mail/access
메일송신을 허용할 IP 및 도메인을 설정합니다.

    Connect:localhost.localdomain           RELAY
    Connect:localhost                       RELAY
    Connect:127.0.0.1                       RELAY
    Connect:mail.devd.kr               RELAY

###### sendmail 서비스 시작
    # service sendmail start
    Starting sendmail:                                     [  OK  ]
    Starting sm-client:                                    [  OK  ]
위에처럼 sendmail 과 sm-client 가 둘다 OK 로 나와야 하는데 sendmail 이 Fail 이 난다면 25번 포트가 충돌되었다고 보면됩니다.

###### 25번 포트 충돌시
smtp 서비스되는 25번 포트를 살펴보면 아래처럼 master라고 나온다. sendmail이 정상 작동이라면 sendmail 이라고 나옵니다. centos 설치하면 기본적으로 postifx가 설치되어있나봅니다.

    # netstat -nlp | grep :25
    tcp        0      0 0.0.0.0:25                  0.0.0.0:*                   LISTEN      68159/master

저처럼 master가 보이신다면 postfix 서비스 정지후 sendmail 재시작 

    # service postfix stop
    # service sendmail restart


추가로 메일 송신시 인증할때 사용하는 데몬도 함께 시작. 저는 이미 설치 되어있더이다.

    # service saslauthd start


###### 서버 재시작시 자동으로 시작하기 위해 추가

    # chkconfig --level 345 saslauthd on
    # chkconfig --level 345 sendmail on


##### dovecot 설치 (IMAP,POP3)

    # yum install dovecot

###### 설정파일 수정 
    # vi /etc/dovecot/dovecot.conf
    // 주석 제거
    protocols = imap pop3 lmtp
    listen = *, ::

plain text 로그인을 위해

    # vi /etc/dovecot/conf.d/10-auth.conf
    disable_plaintext_auth = no

수신 메일 저장 위치 설정

    # vi /etc/dovecot/conf.d/10-mail.conf
    mail_location = mbox:~/mail:INBOX=/var/mail/%u
    //위처럼 설정하면 /home/사용자/mail 로 생성하게 되어 쉘접속없이 메일만 사용하는 계정을 생성한다면 home 폴더가 지저분해지는걸 막기위해 아래처럼 설정합니다.
    mail_location = mbox:/var/empty:INBOX=/var/spool/mail/%u:INDEX=MEMORY
 
그런데 웹메일 사용을 위해 roundcubemail 을 설치해보니 삭제등의 폴더 권한 오류 없이 쓰려면 mbox를 별도 mail 계정 권한의 폴더로 만들어 사용자별 생성해주는것이 좋다. 저도 이부분에서 삽질했는데 결론은 아래처럼 설정하세요.

    mail_location = mbox:/home/mailbox/%u:INBOX=/var/spool/mail/%u:INDEX=MEMORY

/home/mailbox 폴더도 만들어주세요
  
    # mkdir /home/mailbox
    # chown mail: /home/mailbox

###### 추후 roundcubemail 연동시 필요한 설정
    # vi /etc/dovecot/conf.d/20-imap.conf
    //주석해제
    mail_plugins = $mail_plugins


    # vi /etc/dovecot/conf.d/90-plugin.conf
    plugin {
        #setting_name = value
        autocreate = Trash
        autocreate2 = Junk
        autocreate3 = Drafts
        autocreate4 = Sent
        autosubscribe = Trash
        autosubscribe2 = Junk
        autosubscribe3 = Drafts
        autosubscribe4 = Sent
    }

###### dovecot 서비스 시작
    # service dovecot start

###### 재부팅시 자동시작 등록
    # chkconfig --level 345 dovecot on


여기까지 하면 smtp, imap, pop 으로 설정하여 메일을 사용할수 있습니다.

그러나, 우리의 목표는 웹메일 서비스하기!


##### roundcubemail 설치하기

    # wget https://github.com/roundcube/roundcubemail/releases/download/1.2.4/roundcubemail-1.2.4-complete.tar.gz
    # tar zxvf roundcubemail-1.2.4-complete.tar.gz

압축을 푼후 웹메일을 서비스 하기 위해 웹사이트로 운영할 폴더로 이동시킵니다.
 
    # mkdir -p /home/roundcubemail/html
    # mv roundcubemail-1.2.4 /home/roundcubemail/html

roundcubemail 에서 사용할 mysql db 생성

    # mysql -u root -p
    > create database `db_roundcube`;
    > grant all privileges on db_roundcube.* to roundcubeuser@'localhost' identified by 'roundcubepasswd';
    > flush privileges;
    > exit

웹에서 자동 설치하기 위해 config 설정
   
    # cd /home/roundcubemail/html
    # cp config/config.inc.php.example config/config.inc.php
    # vi config/config.inc.php
    $config['enable_installer'] = true;
    //위에서 생성한 디비 정보 입력
    $config['db_dsnw'] = 'mysql://roundcubeuser:roundcubepasswd@localhost/db_roundcube';
    $config['default_host'] = 'devd.kr';


웹에서 구동하기 위해 아파치 설정도 잡아줍니다. (구글링 결과에서는 nginx 설정이 많이 나오네요. 역시 대세는 nginx ㅎㅎ. ) 아래처럼 rewrite 설정도 함께 잡아줍니다.

    # vi /etc/httpd/conf/httpd.conf
    <VirtualHost *:80>
        ServerAdmin cto@devd.kr
        DocumentRoot "/home/roundcubemail/html"
        ServerName webmail.devd.kr
        ErrorLog "logs/webmail.devd.kr-error.log"
        #CustomLog "logs/webmail.devd.kr-access.log" combined
        <Directory "/home/roundcubemail/html">
            RewriteEngine on
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteCond %{REQUEST_FILENAME} !-d
            RewriteRule ^(.+) index.php [L]
         </Directory>
    </VirtualHost>

이렇게 한후 아파치 재시작하고 http://webmail.devd.kr/installer/ 로 들어가시면 웹에서 쉽게 설치가 가능합니다. 설치라기보다 의존성 패키지들을 자동으로 확인해주고 최종 메일 발송테스트까지 해주네요.

여기서 나온 의존성 패키지 설치 안된것들 추가로 설치해줬습니다.

    # pear install Auth_SASL Net_SMTP Net_IDNA2 Mail_mime

Net_IDNA2 이거 설치할때 뭐라뭐라 오류 나는데 그 오류메시지 보고 Net_IDNA2-0.2.0 으로 설치하니까 됩니다.

그리하여 모든것이 통과 됬으면

    # vi config/config.inc.php
    //인스톨 다시 안되게 아래를 주석처리 해줍니다. 그리고 installer 폴더 통째로 삭제하시는게 보안에 좋습니다.라고 하는듯
    //$config['enable_installer'] = true;

오~ 이제 설치하신 http://webmail.devd.kr 로 들어가면 로그인 화면이 나오고 메일계정으로 정상 로그인 됩니다.

그런데.. 웹메일 계정의 패스워드를 못바꾸네요.. ㅎㅎ 제가 사내 직원들의 패스워드를 다 받아서 계정 생성해주면.. 말들이 많아지겠죠.. 그래서 다시 roundcubemail 추가 셋팅

###### roundcubemail password plugin 셋팅
    # vi config/config.inc.php
    $config['plugins'] = array(
      'archive',
      'zipdownload',
      'password' //위 두줄은 원래 있더라고요. 기존건 그대로 두고 password 만 추가
    );

여기서 한참 해맸는데 구글링 하면 password driver 를 `sql` 방식으로만 설명이 되있더라고요. 물론 db로 가상도메인,유저 설정해서 쓰면 될텐데.. 또다시 처음부터삽질을 해야 한다고 생각하니 도저히 못하겠어서 간단하게 `chpasswd` 를 쓰기로 합니다.

    # cp plugins/password/config.inc.php.dist plugins/password/config.inc.php
    # vi plugins/password/config.inc.php
    $config['password_driver'] = 'chpasswd';

이때, 웹서버가 chpasswd 를 root권한으로 사용하기 위해 sudoer에 아래 추가

    # visudo
    apache ALL=NOPASSWD: /usr/sbin/chpasswd

이렇게 하면 아주 쉽게 웹메일에서 사용자가 패스워드를 직접 변경할수 있습니다.

그런데, 뭔가 보안적으로 꺼림직하네요.. 사내 공지하여 초기에 본인이 원하는 패스워드로 갱신하게 하고 sudo 권한은 거두어들이는게 보안상 맞을듯 합니다.

이거 포스트 하는데 생각보다 시간이 많이 걸려서 다음번에 SSL/TLS 설정글 추가할께요

---
***추가팁**: 메일전용계정이라면 /home 폴더에 사용자 계정이 생기지 않게 아래처럼 계정 생성

    # useradd -M help -s /bin/false -g mail
