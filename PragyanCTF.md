2022년 3월 6일에 pragyan CTF를 참여했습니다. 포렌식 문제를 풀어봤으며 한 문제씩 롸업을 한 번 쓰겠습니다.

문제를 한번 보겠습니다.


해석을 잘 하지 못해서 파파고를 써서 해석을 했습니다.

해석 : 프랭크는 한번도 "기술자"가 된 적이 없다. 비밀번호도 다시 쓰고 타이핑도 귀찮고 9 야드나 해요 그리고 이건 기술이 아니야, 그는 박쥐처럼 눈이 멀고 둥근 공만큼 날카로워. 패디로 가는 지름길도 모르잖아 그의 아들 데니스가 기억 보관소를 뒤지고 깃발을 재구성하는 걸 도와줘요

참고: Win7SP1x64 프로필을 사용하여 덤프를 분석하십시오. 이 문제와 관련된 모든 파일은 C 드라이브에만 있고 다른 드라이브에는 없습니다.

 

라는 문구가 쓰여져 있고, MemoryDump라는 이름으로 문제 파일을 하나 주니까 다운을 받아 보겠습니다.

다운을 받으면 zip파일이 하나 있는데, 압축을 바로 풀면 MemDump.DMP 파일이 하나 나옵니다.


그럼 DMP파일을 가지고 메모리 포렌식을 진행해야 하는데, 한번 문제와 같이 읽어보면서 풀어 보겠습니다. 보통 profile을 찾아야 하지만, 해당 문제에서는 Win7SP1x64가 나와 있기 때문에 imageinfo는 생략한다.

 

먼저 프랭크는 한번도 "기술자" 가 된 적이 없다. 비밀번호도 다시 쓰고 타이핑도 귀찮고 << 이 문구를 보아, 프랭크는 타이핑을 치는것을 싫어해서 복사, 붙여넣기 기능을 자주 애용했을 것이라고 추측을 하였다. 그래서 volatility의 플러그인 중 복사한 내용이 담겨있는 저장소가 클립보드이기 때문에, 클립보드를 먼저 찾아 보았다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 clipboard

잘 보면 UNICODETEXT로 포맷된 Data가 하나 있습니다. 딱보니 URL주소이므로 복사해서 들어가보겠습니다.


플래그 값을 찾을 수 있었습니다.

하지만, 1/3이라는 문구로 보아, 3개의 플래그 문자열 중 1개를 찾은겁니다. (괜히 행복했네...ㅠ)

 

이제 두번째 플래그 값을 찾으러 가기 전 기본적으로 이제 미리 분석해야 할 것이 프로세스에서 실행 중인 프로그램을 보는 pslist나 뭐 cmd창에서 뭘 했는지 보는 cmdscan이라던가 등등 한번 찾아봅시다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 pslist
>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 cmdscan
pslist를 써보니 사실 상 의심이 가는 실행 프로그램은 딱히 못봤습니다. 그래서 cmdscan을 했더니 다음과 같은 명령어를 실행한 것으로 보입니다.


C:\Program Files\WinRAR\WinRAR.exe" e comp.rar << 이 명령어를 보아 comp.rar파일을 현재 경로에 압축을 풀겠다는 명령어인데 왠지 심상치 않아 보입니다. 그래서 아까 pslist에서 보니 WinRAR이 실행되고 있는것으로 보아 rar파일을 찾아야 하는데 comp.rar가 있으니 해당 파일을 덤프해서 봐야되겠다는걸 느껴서 메모리 덤프를 바로 했습니다.

 

그 전에 filescan으로 comp.rar가 메모리 주소 어디에 있는지 알아야 하니까 메모리 주소먼저 찾아보겠습니다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 filescan > scan.txt
이 명령어는 filescan 플러그인을 사용해서 나오는 값을 scan.txt에 데이터를 옮겨라 이 뜻입니다.

그럼 현재 경로에 scan.txt 파일이 하나 생기는데 comp.rar를 찾아 보겠습니다.


메모리 주소는 0x000000003df4e450 임을 확인했으니 바로 파일 덤프를 뜨겠습니다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 dumpfiles -Q 0x000000003df4e450 -D ./
해당 명령어를 사용하시면 다음과 같이 2개의 comp.rar 파일을 복구합니다.


파일을 추출하면 .dat이나 .vacb같은 원래 확장자가 아닌 다른 확장자로 추출이 되는데, comp.rar로 이름을 바꿔주고 압축을 풀어봅시다.



압축을 풀려하니...비밀번호가 걸려있네요...그래서 rarcracker로 2시간을 돌려봤지만 답을 구할 수 없었습니다.

그래서 고민을 해보니까 문제에 프랭크...프랭크...그리고 추출을 할 comp.rar파일에 경로를 보면 유저가 Frank Reynolds라고 되어있음을 알 수 있습니다. 그래서 사용자의 암호 해쉬값을 찾기 위해서 hashdump를 사용 하겠습니다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 hashdump


사용자의 암호 해쉬값을 해제 하니까 trolltoll이라는 비밀번호를 찾을 수 있었습니다. 그래서 압축을 푸니 2번째 플래그 값을 구할 수 있었습니다.


 

마지막 플래그를 찾기 위해서는 이 문장을 이해 하셔야 합니다.

>> 패디로 가는 지름길도 모르잖아 그의 아들 데니스가 기억 보관소를 뒤지고 깃발을 재구성하는 걸 도와줘요

 

쉽게 말해서 Win7SP1x64 프로필을 사용하는데, 지름길 이라는것은 lnk파일을 생각할 수 있을것이다.

윈도우 7에서는 lnk파일로 바로가기를 지정하여 특정 프로그램을 실행시킬 수 있습니다.

그래서 lnk를 찾아야 하니까 아까 위에서 filescan으로 파일의 목록들을 추출했으니 추출한 텍스트 파일로 가서 lnk를 입력해봅시다.


Frank Reynolds 유저에서 Paddys.lnk를 찾을 수 있었습니다. 문제에서 "패디로 가는 지름길도 모르잖아" 이 부분이 파일명을 뜻하는거 같습니다.

 

그래서 덤프파일을 추출하겠습니다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 dumpfiles -Q 0x000000003e1891d0 -D ./
추출을 하게 되면 다음과 같이 sysinfo.txt를 가리키는 것을 볼 수 있습니다. 


하지만 이것을 보고는 추측할 수 있는게 덤프되어 삭제되었으니 파일이 깨져있구나...라고 생각을 했었습니다. 그래서 파일 시스템을 알아보기 위해 hxd에디터로 FAT NTFS 등등 검색을 해보니 NTFS로 파일 시스템이 포맷되었음을 알 수 있었습니다.


NTFS는 덤프되어 삭제된 파일을 가지고 오기 위해서는 $MFT에서 파일을 가져와야 합니다. 그래서 volatility 플러그인 중 mftparser를 이용해서 가져오겠습니다. 저는 깔끔하게 하기 위해서 csv파일로 가져왔습니다.

>> volatility2.6.exe -f MemDump.DMP --profile=Win7SP1x64 mftparser > mft.csv

이렇게 삭제된 경로랑 똑같은 파일을 볼 수 있습니다. 그래서 찾아가봤더니 마지막 플래그 값을 구할 수 있었습니다.


이렇게 하나의 문제를 풀어봤는데...설명을 제대로 한거같지는 않지만 그래도 나름 열심히 설명했다고 생각합니다.

 

FLAG : p_ctf{v0l4t1l1ty_i5_v3ry_h4ndy_at_r34d1ng_dump5_iasip}


해결 완료!!!!!!!!!!!!!!!!!!!!
