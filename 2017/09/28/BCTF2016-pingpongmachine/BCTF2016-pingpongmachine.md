---
title: BCTF2016 pingpongmachine
date: 2017-09-28 16:49:43
tags: 
 - android
---

#  pingpong - 555

拿到题之后解包, 整体上Java逻辑没有明显混淆, C层用了O-LLVM混淆, 但是代码量不大. 整体上没有加壳或者对完整性校验. 通过观察Java层逻辑, 要求我们依次点击ping pong两个按键, 共计100万次.

```Java
    public MainActivity() {
        super();
        this.p = 0;
        this.num = 0;
        this.ttt = 1000000;
        this.tt = this.ttt;
        this.jping = new com.geekerchina.pingpongmachine.MainActivity$1(this);
        this.jpong = new com.geekerchina.pingpongmachine.MainActivity$2(this);
    }

        public void onClick(View arg7) {
            if(MainActivity.this.tt % 2 == 1) {
                MainActivity.this.p = 0;
                MainActivity.this.num = 0;
                MainActivity.this.tt = MainActivity.this.ttt;
            }

            --MainActivity.this.tt;
            MainActivity.this.p = MainActivity.this.ping(MainActivity.this.p, MainActivity.this.num);
            ++MainActivity.this.num;
            if(MainActivity.this.num >= 7) {
                MainActivity.this.num = 0;
            }

            View v0 = MainActivity.this.findViewById(2131427414);
            ((TextView)v0).setText("PING");
            if(MainActivity.this.tt == 0) {
                ((TextView)v0).setText("FLAG: BCTF{MagicNum" + Integer.toString(MainActivity.this.p) + "}");
            }
        }

        public void onClick(View arg7) {
            if(MainActivity.this.tt % 2 == 0) {
                MainActivity.this.p = 0;
                MainActivity.this.num = 0;
                MainActivity.this.tt = MainActivity.this.ttt;
            }

            --MainActivity.this.tt;
            MainActivity.this.p = MainActivity.this.pong(MainActivity.this.p, MainActivity.this.num);
            ++MainActivity.this.num;
            if(MainActivity.this.num >= 7) {
                MainActivity.this.num = 0;
            }

            View v0 = MainActivity.this.findViewById(2131427414);
            ((TextView)v0).setText("PONG");
            if(MainActivity.this.tt == 0) {
                ((TextView)v0).setText("FLAG: BCTF{MagicNum" + Integer.toString(MainActivity.this.p) + "}");
            }
        }
```

在C层会发现sleep(1) 暂停一秒, 如果直接调用点击时间肯定不够, 因此将sleep一秒patch掉, 可以改为 mov r1, r1.
之后自己写一个包名和原程序相同的包, 依次调用ping 和pong 共计100万次, 随后输出结果.

一个示例的调用程序为:

```Java
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        int p = 0;
        int num = 0;

        p = pong(p, num);
        num ++;
        for(int tt = 1000000 - 1; tt != 0; --tt){
            if(tt % 2 == 0){
                p = ping(p,num);
                num ++;
                if(num >=7) num = 0;
            }else{
                p = pong(p, num);
                num ++;
                if(num >= 7) num = 0;
            }
        }
        //Log.e("yourdad", Integer.toString(p));
        ((TextView)findViewById(R.id.a)).setText(String.valueOf(p));
```