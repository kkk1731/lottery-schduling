# lottery-schduling

## 実装

### proc.h
ここではproc構想隊に必要な情報を追加する。

tickets：プロセスが持つチケット枚数
(以下動作確認・デバッグ用)
called_times：プロセスが呼ばれた回数
winner：lottery schedulingで当選した番号
sum_tickets：RUNNABLEなプロセスの合計チケット数

lottery.h内　proc構造体
```
struct proc {
 ・・・
  int called_times;
  int tickets;
  int winner;
  int sum_tickets;
}; 

```

### proc.c
実際のスケジューリング部分の実装を行う部分。

#### include
```
#include "rand.h"
```
上記のrand.hを宣言する。rand.hは、新たに追加した乱数を生成するためのファイル。後述する。

#### allocproc 関数
プロセスが生成されるときに用いられる関数であるため、ここでプロセスのチケット枚数や呼ばれた回数(この時点では0)も初期化する。
なお p は生成したプロセスの構造体のポインタ。( struct proc *p)
```
・・・
found:
  p->pid = allocpid();
  p->state = USED;
  p->tickets = 100;
  p->called_times = 0;
  ・・・
  ```
  
  #### scheduler 関数
  スケジューリングを行う関数。もともとはラウンドロビンスケジューリングが実装されていたが、これに追加・編集してlotteryを実装する。
  
  まずcount_ticket関数でRUNNABLEなプロセスのチケット総数(=sum_tick) を計算する。
  次に 0以上sum_tick未満の乱数を生成し、これを当選したチケット (winner) とし、以後for文で当選したプロセスを探して実行している。
  また選ばれたプロセスの called_timesは1インクリメントし、確認用にwin_tick, sum_tick も更新している。
  
  ```
  
  scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  int win_tick;
  int sum_tick;
  int count;

  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    sum_tick = count_tickets();
    win_tick = random_at_most(sum_tick);
    count = win_tick;

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
	if(p->tickets > count){
          // Switch to chosen process.  It is the process's job
          // to release its lock and then reacquire it
          // before jumping back to us.
          p->state = RUNNING;
          c->proc = p;
          swtch(&c->context, &p->context);
	  p->called_times += 1;
	  p->sum_tickets = sum_tick;
	  p->winner = win_tick;
          // Process is done running for now.
          // It should have changed its p->state before coming back.
          c->proc = 0;
	  release(&p->lock);
	  break;
        }
	count -= p->tickets;
      }
      release(&p->lock);
    }
  }
}

```

### rand.c rand.h
メルセンヌツイスターで乱数を生成するためのファイル。
こちらから引用させてもらいました。https://github.com/avaiyang/xv6-lottery-scheduling/blob/master/rand.c
線形合同法で乱数生成を自作することも考えたが、時間の都合上諦めた。
とりあえず random_at_most(long max) 関数で0以上max未満の乱数を生成できる。

### 動作確認
下のプログラムで、親プロセスと子プロセスを平行実行させて確認した。それぞれチケットを20，10まいずつ配り、約10秒間並行実行させた。
その後当選番号、合計チケット数、チケット保持数、呼ばれた回数を出力する。

結果は次のようになった。チケット枚数と呼び出された回数の比がおおよそ一致しており、parentは当選番号0~19、childは20~29で選ばれている(正確には分からないが)と考えられるので、ただしく実装できたと考えられる。

#### 結果
```
$ ./test_lottery

parent
winner = 2
sum tickets=30
my tickets=20
calledtimes=68
$ 
child
winner = 24
sum tickets=30
my tickets=10
calledtimes=37
```

#### コード
```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[]){
	int pid;
	int start, end;
	start = uptime();
	if((pid = fork()) == 0){
		change_tickets(10);
		while(1){
			end = uptime();
			if((end-start) > 100)
				break;
		}
		printf("\nchild\nwinner = %d\nsum tickets=%d\nmy tickets=%d\ncalledtimes=%d\n", return_winner(), return_sum_tickets(), return_tickets(), return_called_times());
		exit(1);

	} else if(pid > 0){
		change_tickets(20);
		while(1){
			end = uptime();
			if((end-start) > 100)
				break;
		}
		printf("\nparent\nwinner = %d\nsum tickets=%d\nmy tickets=%d\ncalledtimes=%d\n", return_winner(), return_sum_tickets(), return_tickets(), return_called_times());
		exit(1);
		
	} else {
		exit(1);
	}

	wait(0);
	exit(1);
}

```

