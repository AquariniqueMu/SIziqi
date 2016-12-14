#include <Wire.h>
//#include"music.h"
#include "U8glib.h"
#include<Microduino_Key.h>
#include <avr/sleep.h>
U8GLIB_SSD1306_128X64 u8g(U8G_I2C_OPT_NONE);//设置屏幕
Key KeyA(A0,INPUT);
Key KeyB(A0, INPUT);
Key KeyC(A0, INPUT);
Key KeyD(A0, INPUT);
Key KeyE(A0, INPUT);
const int touchpadPIN=2;
void setup() {
  Wire.begin();        // 加入i2c总线，作为主机
  Serial.begin(9600);  
  pinMode(touchpadPIN,INPUT);
//  pinMode(buzzer_pin, OUTPUT);
}
class boarddata
{
public:
  int board[6][7];//6行7列
              //0 没有 1 自己 2 敌方
   boarddata()//由于microduino不能列表初始化，故不得不用memset
   {          //只能自己手写一个构造函数
    memset(board,0,sizeof(board));
   }
  bool action_me(int row)//到时候再换成byte, line 行，row 列
  {
    if (row < 0 || row>6) return false;
    for (int i = 0; i < 6; ++i)
    {
      if (board[i][row] == 0)
      {
        board[i][row] = 1;
        return true;
      }
    }
   // cout << "error" << endl;
      return false;
  }
  bool receive(int row)
  {
    if (row < 0 || row>6) return false;
    for (int i = 0; i < 6; ++i)
    {
      if (board[i][row] == 0)
      {
        board[i][row] = 2;
        return true;
      }
    }
  //  cout << "error" << endl;
    return false;
  }
  bool check_2(int line, int row, int number,int player)//  查询 \形状
          //当前在 board[line][row],现在一共有number个连成 \ 线
  {
    if (number == 4) return true;
    if (line + 1 <= 6 && row - 1 >= 0&&board[line+1][row-1]==player)
      if(check_2(line + 1, row - 1, number + 1,player)) return true;
      else return false;
      return false;
  }
  
  bool check_3(int line, int row, int number,int player)//  查询 /形状
                        //当前在 board[line][row],现在一共有number个连成 / 线
  {
    if (number == 4) return true;
    if (line + 1 <= 6 && row +1 <= 7&&board[line+1][row+1]==player)
      if (check_3(line + 1, row + 1, number + 1,player)) 
        return true;
      else 
        return false;
    return false;
  }
  bool check_four_in_line(int player)//dfs,player中1是主机，2是从机
  {       
    for(int line=0;line<=6;++line)
      for (int row = 0; row <= 7; ++row)
      {
        if (board[line][row] == player)
        {
          if (row <= 4)
          {
            auto &templine = board[line];
            if (templine[row + 1] == player && templine[row + 2] == player && templine[row + 3] == player)
              return true;
          }
          if (line <= 3)
          {
            if (board[line + 1][row] == player && board[line + 2][row] == player
                      && board[line + 3][row] == player)
              return true;
          }
          if (this->check_2(line, row, 1,player)) return true;
          if (this->check_3(line, row, 1,player))return true;
        }
      }
    return false;
  }
  
  /*  c++方案
  void print_board()
  {
    for (int line = 5; line >= 0; --line)
    {
      for (int row = 0; row <= 6; ++row)
        cout << board[line][row] << " ";
      cout << endl;
    }
  }*/
  void draw_board_point (int line,int row)
  {
  if(board[line][row]==1)//我方棋子，用实心圆
                              //u8g上，x坐标控制列，y坐标控制行
    u8g.drawDisc(2+row*19,57-11*line,2);
   else if(board[line][row]==2)//敌方棋子，空心圆
    u8g.drawCircle(2+row*19,57-11*line,2);
  }
  void print_board()//用u8g实现图像化，只采用60*126的屏幕大小
  {                  //经测试，左上角为0,0
    u8g.firstPage();
    do
    {
      for(int y=2;y<=62;y+=11)
        u8g.drawHLine(2,y,115);
      for(int x=2;x<=124;x+=19)
       u8g.drawVLine(x,2,56);
      //u8g.drawDisc(2,57,2);  不同点的具体坐标请看  zuobiao.txt
      for(int line=0;line<6;++line)//开始画棋子
        for(int row=0;row<7;++row)
          if(this->board[line][row]!=0)//有棋子，但是得交给下面函数判断敌我棋子
            draw_board_point(line,row);
    }while(u8g.nextPage());
  }
  void print_board(const int row)//该重载函数是在正在输入时调用
  {
    u8g.firstPage();
    do
    {
      for(int y=2;y<=62;y+=11)
        u8g.drawHLine(2,y,115);
      for(int x=2;x<=124;x+=19)
       u8g.drawVLine(x,2,56);
      //u8g.drawDisc(2,57,2);  不同点的具体坐标请看  zuobiao.txt
      for(int line=0;line<6;++line)//开始画棋子
        for(int row=0;row<7;++row)
          if(this->board[line][row]!=0)//有棋子，但是得交给下面函数判断敌我棋子
            draw_board_point(line,row);
       //上方代码已经打完了已有的棋子，接下来打印正在输入的棋子
        if((millis()/500)%2==0)
          u8g.drawDisc(2+19*row,2,2); 
    }while(u8g.nextPage());
  }
}; 
boarddata my_board;
void loop() {
  /*测试代码
   Wire.requestFrom(8, 6);    // 通知8号从机上传6个字节
 
  while (Wire.available()) { 
    char c = Wire.read(); 
    Serial.print(c);         
  }
  Wire.beginTransmission(8);
  Wire.write(2);
   Wire.endTransmission();
  delay(500);
  */
  
  int row=0;
  //这里进行输入自己的走法,走完后打印棋盘,row此时已经被赋值
  input_screen(row);
  my_board.action_me(row);
  my_board.print_board();
  bool winflag=my_board.check_four_in_line(1);
  //if(winflag) winner();//用于调试
  //Serial.println("chongxinjinru loop");
  /*
  Wire.beginTransmission(8);
  Wire.write(winflag);
  Wire.write(row);
  Wire.endTransmission();
  */
  if(winflag)
  {
    end_screen(1);//进入到胜利界面，并加入中断
    return;
  }
  
  bool input_confirm=false;
  int enemy_row=0;
byte temp_row=0;
  while(true)
  {
    Wire.requestFrom(8,1);//第一个：用来传输temp_row,第二个:是否确定输入
    while(Wire.available()>0)
           temp_row=Wire.read();//注意！传输过来的temp_row是被压缩后的数据
     //解压过程
     input_confirm=temp_row%10;
     temp_row/=10;
    my_board.print_board(temp_row);
    Serial.print(temp_row);
    Serial.print(" ");
   Serial.println(input_confirm);
    if(input_confirm) 
    {
      enemy_row=temp_row;
      break;
    }
  }
  my_board.receive(enemy_row);
  my_board.print_board();
 // delay(2000);//让敌人知道怎么输的！哈哈哈
 bool enemy_winflag=my_board.check_four_in_line(2);
  if(enemy_winflag)  
  {
    delay(2000);
    end_screen(0);//进入失败界面
  }
 // delay(1000);
   
   /**********测试代码*******
  char serialin='0';
  while(serialin=='0')
  {
  while(Serial.available()>0)
    serialin=Serial.read();
  Serial.println(serialin);
  my_board.print_board();
  }
  delay(10000);
  Serial.println("exit");
  ****************************/
 
}
void input_screen(int &row)//只有左右
{
  while(KeyE.read(0,20)!=SHORT_PRESS&&KeyE.read(0,20)!=LONG_PRESS)
  {
    my_board.print_board(row%7);
    switch (KeyB.read(480, 530)) {//向左
    case SHORT_PRESS:
      Serial.println("KEY LEFT(analog) SHORT_PRESS");  //短按
      --row;
      if(row<0) row+=7;
      my_board.print_board(row%7);
      break;
    case LONG_PRESS:
      Serial.println("KEY LEFT(analog) LONG_PRESS");    //长按
      break;
  }
  switch (KeyC.read(830, 870)) {//向右
    case SHORT_PRESS:
      Serial.println("KEY RIGHT(analog) SHORT_PRESS");  //短按
        ++row;  
        my_board.print_board(row%7);
      break;
    case LONG_PRESS:
      Serial.println("KEY RIGHT(analog) LONG_PRESS");    //长按
      break;
  }
  }
}
void end_screen(const int &flag)
{
    u8g.firstPage();
  do
  {
    u8g.setFont(u8g_font_unifont);
    if(flag==1)
    u8g.drawStr(21,35,"Zhuji Win!!");
    else 
     u8g.drawStr(21,35,"Congji Win!!");
  }while(u8g.nextPage());
  memset(my_board.board,0,sizeof(my_board.board));
  attachInterrupt(0,zhongduan,LOW);
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);
    sleep_enable();
    sleep_mode();
    sleep_disable();
}
void zhongduan()
{
  Serial.println("xinglai");
   detachInterrupt(0);
}
