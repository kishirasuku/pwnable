# pwnable collision

pwnable.krの"collision"のWriteupです

以下が問題文です
>Daddy told me about cool MD5 hash collision today.
>I wanna do something like that too!

ハッシュの衝突とは、異なる入力で同様の出力がでてくることを言うようです。
問題を解いてみた感想としてはこれがハッシュの衝突といいきれるのかは分かりませんでした。予想としてはcheck_password関数を１つのハッシュとしてみていてそれの衝突なのかなという風に感じました。

以下は問題となるCプログラムです。

```
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

この問いでは、標準入力に20byte持たせて実行させるプログラムになっています。
まずは

```
col@pwnable:~$ ./col $(python -c 'print "A"*20')
```
として実行させるやり方で考えてみました。

check_passwordでは文字列を入力させ、それをint型に変換させて考えています。つまり1byteから4byteごとの入力にされるわけですね。そして20byteの標準入力なのでfor文を5回回して確かにちょうでですね。

ここでgdbデバッグをしてみた結果を一部載せます。

```
(gdb) p ip[0]
$8 = 1094795585
```

```
(gdb) x/4x &ip[0]
0x7fffffffe455:	0x41	0x41	0x41	0x41
```

ここでip[0]の結果を見てみるとresに1094795585が加算されています。これがなぜかを考えていきましょう。

そこでx/4x &ip[0]の結果を見ていくことになります。
"A"を20個入力したわけですが、"A"一つ文は65、16進数では0x41となります。これをint型で表されると4byteでみなされるのでip[0]の中身は0x41414141、つまり1094795585となるわけです。

ここまでわかったので、flagに到達するにはipを用いて五回に分けてresに加算し0x21DD09ECにさせてあげればいいわけですね。

0x21DD09ECは10進数では568134124
568134124=113626824*4+113626828で考えられますね。

それぞれの項のリトルエンディアンを考えて入力させると

```
col@pwnable:~$ ./col $(python -c 'print "\xc8\xce\xc5\x06"*4+"\xcc\xce\xc5\x06"')
daddy! I just managed to create a hash collision :)
```

