---
title: C++ 学生管理系统
published: 2022-07-09
description: "初学者练习"
tags: [C++, 初学]
toc: false
---

一道非常经典的C语言题目，用C++实现

题目如下：
  - 输入功能：由键盘输入10个学生的学号、姓名、三科成绩，并计算出平均成绩和总成绩，然后将它存入文件stud.dat。
  - 插入功能：按学号增加一个学生信息，并将其插入到stud.dat中。
  - 排序功能，按要求对学生信息进行排序，分为按学号和按总成绩进行排序两种情形，并输出结果。
  - 查询功能：按要求查找学生信息，分为按学号和姓名进行查询两种情形，并输出结果。
  - 删除功能：按要求将学生信息删除，分为按学号和姓名进行删除两种情形。
  - 输出功能：按学号输出学生信息。


## 整体思路

- 程序启动的时候判断文件(stu.dat)是否存在，如果文件不存在，则正常执行，如果文件存在，先获取文件中学生的个数，根据学生的个数创建对象数组，将内容创建成学生对象，保存在对象数组1里，再向下执行。

- 用Switch语句来判断不同的输入。

- 新增学生，根据 原来对象数组1储存的人数+新增的人数 来确定新的动态数组2的大小，将原本对象数组1内的内容保存在新的对象数组2里，再将新增的内容储存在后面，每次新增完，直接保存到文件。

- 排序学生，根据学号或者姓名，写一个数组的冒泡排序即可

- 查询学生，写一个函数，判断学生是否存在，如果存在返回学生所在数组的下标，根据下标输出内容

- 删除学生，用查询学生写的函数，根据下标删除学生

- cout对象数组里的内容就完事


## 代码实现

```C++

#include <iostream>
#include<fstream>
#include<string>
#define line for (int n = 0; n <= 100; n++) cout << "-"
#define FILENAME "stdu.dat"

using namespace std;
class student { //学生类
public:
    int Is_Exist(string stuId, int a);
    int get_student_number(); //获取文件中学生人数
    bool File_Is_Empty; //文件是否为空的标识
    student *studentArray; //将文件中的内容，以student对象方式储存在studentArray[]数组中
    int student_number; //学生人数
    string studentId; //学号
    string name;   //姓名
    float score[3];//成绩*3
    float Total;    //    总分
    int Average;//平均分

    void sort(int n=1); //排序函数
    void delete_stu(); //删除学生
    void init(); //初始化内容，将文件中的内容读到studentArray中
    void save(); //保存文件
    void show();//展示界面
    void add_stu(int number = 1); //添加学生
    void showInfo(); //展示学生信息
    void search();//搜索学生
    student() //默认构造函数，判断文件是否为空，设置File_Is_Empty值
    {
        ifstream ifs;
        ifs.open(FILENAME, ios::in);
        if (!ifs.is_open())
        {
            this->student_number = 0;
            this->studentArray = NULL;
            this->studentId = "0";
            this->name = "0";
            this->score[0] = 0;
            this->Average = 0;
            this->Total = 0;
            this->File_Is_Empty = true;
            ifs.close();
            return;
        }
        char c;
        ifs >> c;
        if (ifs.eof())
        {
            //文件空
            this->student_number = 0;
            this->studentArray = NULL;
            this->studentId = "0";
            this->name = "0";
            this->score[0] = 0;
            this->Average = 0;
            this->Total = 0;
            this->File_Is_Empty = true;
            ifs.close();
            return;
        }
        int num = this->get_student_number(); //获取文件中学生数量
        this->student_number = num;
        //cout << "现在学生人数为:" << num << endl;;
        //system("pause");
    };
    ~student() //析构函数
    {
        delete[]this->studentArray;
        this->studentArray = NULL; //防止指针变为野指针
    }
    student(string stuId,string stuName, float stuScore[3]) //带参数的构造函数
    {
        this->studentId = stuId;
        this->name = stuName;
        for (int i = 0; i < 3; i++)
            this->score[i] = stuScore[i];
        this->Average = (stuScore[0] + stuScore[1] + stuScore[2]) / 3;
        this->Total = stuScore[0] + stuScore[1] + stuScore[2];
    };

};
void student::sort(int n) //排序，当n=1的时候为按学号排序，n=2为按总成绩排序
{
    int num1; //学号在设计的时候是string类型，用int类型排序需要atoi，用num接收转换过的值
    int num2; //
    student a; //用于交换对象数组的中间变量

    if (this->File_Is_Empty) //判断文件是否为空
    {

        cout << "数据文件不存在或者为空\n";
        system("pause");
    }
    else
    {
        cout << "3.学生信息排序\n";
        if (n == 1)
        {
            cout << "将学生信息按学号顺序排序\n";
            for (int i = 0; i < this->student_number - 1; i++)//冒泡排序 
            {
                for (int t = 0; t < this->student_number - 1 - i; t++)
                {
                    num1 = atoi(this->studentArray[t].studentId.c_str()); //将string类型变量变为int类型
                    num2 = atoi(this->studentArray[t + 1].studentId.c_str());
                    if (num1 > num2)
                        a = this->studentArray[t + 1], this->studentArray[t + 1] = this->studentArray[t], this->studentArray[t] = a;
                }
            }
        }
        if (n == 2)
        {
            cout << "将学生信息按总成绩顺序排序\n";
            for (int i = 0; i < this->student_number - 1; i++)//冒泡排序 
            {
                for (int t = 0; t < this->student_number - 1 - i; t++)
                {

                    if (this->studentArray[t].Total > this->studentArray[t + 1].Total)
                        a = this->studentArray[t + 1], this->studentArray[t + 1] = this->studentArray[t], this->studentArray[t] = a;
                }
            }
        }
    }
}
void student::search() //查找学生成绩
{
    string id;
    int ret;   //studentArray[]的下标,在Is_Exist()中
    int a = 1; //判断根据学号查找还是姓名查找
    cout << "4.查找学生成绩\n" << "1.按照学号查找\t2.根据姓名查找\n输入你的选项>";
    cin >> a; 
    if (a == 1)
    {
        cout << "请输入学号:", cin >> id;
        ret = this->Is_Exist(id, 1);
    }
    else if (a == 2)
    {
        cout << "请输入姓名:", cin >> id;
        ret = this->Is_Exist(id, 2);
    }
    else
    {
        ret = this->Is_Exist(id, 1);
    }
    if (ret == -1)
    {
        system("cls");
        cout << "学生不存在" << endl;
    }
    else
    {
        cout << "查找成功，下面为该学生信息\n" << endl;
        cout << "学号:" << this->studentArray[ret].studentId
            << " 姓名:" << this->studentArray[ret].name << endl
            << "各门成绩:语文:" << this->studentArray[ret].score[0]
            << " 数学:" << this->studentArray[ret].score[0]
            << " 英语:" << this->studentArray[ret].score[2]
            << " 平均成绩:" << this->studentArray[ret].Average
            << " 总成绩:" << this->studentArray[ret].Total << endl;
    }
    system("pause");
    
}
int student::Is_Exist(string stuId,int a)
{
    //a=1 则使用学生学号查找
    //a=2 则使用学生姓名查找
    if (File_Is_Empty) //判断文件是否不存在或者为空，如果为空则返回-2
    {
        cout << "数据文件不存在或者为空\n";
        system("pause");
        return -2; 
    }
    for (int i = 0; i < this->student_number; i++) 
    {
        if ((a==1)&&(this->studentArray[i].studentId == stuId)) //根据学号查找
        {
            return i;
        }
        if ((a == 2) && (this->studentArray[i].name == stuId)) //根据姓名查找
        {
            return i;
        }
    }
    return -1; //如果没找到返回-1
}
void student::delete_stu() //删除学生
{
    int ret; //接收 Is_Exist的返回值，返回值是此学生在studentArray[]里的下标
    string id; //可以用来接收学号或者姓名
    int a = 1; //判断是按照学生学号寻找还是根据学生姓名寻找
    cout << "5.删除学生成绩\n"
        << "1.按照学号删除\t2.根据姓名删除\n输入你的选项>";
    cin >> a;
    if (a == 1) //按照学生学号查找
    {
        cout << "请输入学号:", cin >> id;
        ret = this->Is_Exist(id, 1);
    }
    else if(a==2)//按照学生姓名
    {
        cout << "请输入姓名:", cin >> id;
        ret = this->Is_Exist(id, 2);
    }
    else  //默认按照学生学号进行查找
    {
        cout << "请输入学号:", cin >> id;
        ret = this->Is_Exist(id, 1);
    }
    if (ret == -1) //返回值为-1则学生不存在
    {
        system("cls");
        cout << "学生不存在"<<endl;
    }
    else if(ret == -2) //返回值为-2文件不存在或为空，直接退出函数
    {
        return;
    }
    else //如果学生信息存在，并且ret是studentArray[]中的下标
    {//输出要删除的学生信息
        cout << "该学生的信息将被删除"<<endl;
        cout << "学号:" << this->studentArray[ret].studentId
            << " 姓名:" << this->studentArray[ret].name << endl
            << "各门成绩:语文:" << this->studentArray[ret].score[0]
            << " 数学:" << this->studentArray[ret].score[0]
            << " 英语:" << this->studentArray[ret].score[2]
            << " 平均成绩:" << this->studentArray[ret].Average
            << " 总成绩:" << this->studentArray[ret].Total << endl;
        for (int i = ret; i < this->student_number - 1; i++)
        {
            this->studentArray[i] = this->studentArray[i + 1];
        }
        this->student_number--;
        this->save();
    }
    system("pause");
}
void student::init()
{
    string name;       //用来储存读取到的姓名
    string student_id;//用来储存读取到的学号
    int i = 0;            //标识符，用来把实例化的类储存到对应的对象数组
    float score[3];        //用来储存读取到的成绩
    float total;        //用来储存读取到的成总分
    int average;        //用来储存读取到的平均分
    ifstream inf;        //实例化一个文件对象
    this->studentArray = new student[this->student_number]; //根据读取到的学生数量开辟空间，学生数量
    inf.open(FILENAME, ios::in);
    while (inf >> student_id && inf >> name && inf >> score[0]&& inf >> score[1] && inf >> score[2] && inf >> average && inf >> total)
    {        //格式化读取文件,从文件读到变量
        student stu(student_id, name, score); //使用读到的值实例化student对象
        this->studentArray[i] = stu; //把实例化的对象放到stu1对象的studentArray中
        i++;
    }

}
int student::get_student_number() //获取文件中学生的数量，在add_stu()函数中加上需要增加的学生数量，为最新需要开辟的空间的大小
{ 
    string name;
    string student_id;
    int num=0;  //每格式化读取一块数据，则加1人数
    float score[3];
    float total;
    int average;
    ifstream inf;
    inf.open(FILENAME, ios::in);
    while (inf >> student_id && inf >> name && inf >> score[0] && inf >> score[1] && inf >> score[2] && inf >> average && inf >> total)
    {
        num++;
    }
    inf.close();
    return num; //返回学生数量
}
void student::save() //将stu1中studentArray中的每个student对象储存到文件中
{
    ofstream ofs;
    ofs.open(FILENAME,ios::out);
    for (int i = 0; i < this->student_number; i++)
    {
        ofs << this->studentArray[i].studentId << " " 
            << this->studentArray[i].name << " " 
            << this->studentArray[i].score[0] << " " 
            << this->studentArray[i].score[1] << " " 
            << this->studentArray[i].score[2]<< " "
            <<this->studentArray[i].Average<<" "<< this->studentArray->Total<<" ";
    }
}
void student::showInfo() //显示学生信息
{
    if (this->File_Is_Empty) //构造函数里的文件标识符，判断文件是否存在或者为空，每实例化一个对象都检查一遍
    {
        system("cls");
        cout << "文件不存在或内容为空,请先输入数据\n";
    }
    else
    { //文件不为空则输出stu1中studentArray中储存的内容
        for (int i=0;i<this->student_number;i++)
        {
            cout << "学号:" << this->studentArray[i].studentId
                << " 姓名:" << this->studentArray[i].name << endl
                << "各门成绩:语文:" << this->studentArray[i].score[0]
                << " 数学:" << this->studentArray[i].score[1]
                << " 英语:" << this->studentArray[i].score[2]
                << " 平均成绩:" << this->studentArray[i].Average
                << " 总成绩:" << this->studentArray[i].Total << endl;
            line;
            cout << endl;
        }
    }
    system("pause");
}
void student::add_stu(int add_number) //添加学生
{
    //计算需要的新的空间大小
    int newsize = this->student_number + add_number; //需要开辟的空间 = 文件中学生的人数+需要添加的数量
    //cout << "newxize=" << newsize;
    student *newspace = new student[newsize]; //开辟内存空间
    student *stu3;
    string name;
    string studentId;
    float score[3];
    if (this->studentArray!= NULL) //如果studentArray不为空就先将studentArray中的内容先复制到newspace数组中，再在newspace中增加学生信息,
                                        //最后再将newspace的内容复制到studentArray中，实现动态数组
    {
        for (int i = 0; i < this->student_number; i++)
        {
            newspace[i] = this->studentArray[i];
        }
    }
    for (int i = 0; i < add_number; i++) //输入学生信息
    {
        cout << "请输入学生姓名:", cin >> name, cout << endl;
        cout << "请输入学生学号:", cin >> studentId, cout << endl;
        cout << "请输入学生语文成绩:", cin >> score[0], cout << endl;
        cout << "请输入学生数学成绩:", cin >> score[1], cout << endl;
        cout << "请输入学生英语成绩:", cin >> score[2], cout << endl;
        line;
        cout << endl;
        student stu3(studentId, name, score); //将输入的内容实例化成对象
        newspace[this->student_number + i] = stu3; //将对象依次储存到newspace中
    }
    delete[] this->studentArray;
    this->studentArray = newspace; //将newspace赋给studentArray，用来在别的成员函数中访问
    this->student_number = newsize; //更新学生人数大小
    this->File_Is_Empty = false; // 输入了内容以后，文件不为空
    this->save(); //保存到文件中
    cout << "添加成功"<<endl;
    
    system("pause");

}
void student::show()
{
    cout << "1.输入学生成绩" << endl << "2.增加学生成绩" << endl << "3.学生信息排序"
        << endl << "4.查找学生成绩" << endl << "5.删除学生成绩" << endl << "6.显示学生成绩"
        << endl << "7.安全退出系统" << endl;
    line;
    cout << endl;
    cout<< "输入你的选择>:";

}
student stu1; //实例化对象，此时已经得到文件中的学生数量
int main()
{
    int choice=7;
    
    stu1.init(); // 将文件里的内容按照格式读入内存，储存到
//    cout << "stu1里的人数" << stu1.student_number;
//    system("pause");
    while (true)
    {
        system("cls");
        stu1.show();
        cin >> choice;
        line;
        cout << endl;
        
        if (cin.good() && choice <= 7 && choice >= 1)  //判断用户输入是否合法
        {
            switch (choice)
            {
            case 1: 
                //添加学生功能，默认参数为1，用来增加学生时直接调用，初始化添加十个学生时传递参数10
                stu1.add_stu(10);
                break;
            case 2:
                //增加学生，要求按照学号顺序插入,则在增加学生后调用排序函数，再进行保存
            {
                stu1.add_stu();
                stu1.sort();
                stu1.save();
            }
                break;
            case 3:
                //排序功能
            {
                int n = 1;
                cout << "1.按照学号排序\t2.按照总成绩排序\n";
                cin >> n;
                stu1.sort(n);
                stu1.showInfo();
            }
                    break;
            case 4:
                //搜索功能
                stu1.search();
                break;
            case 5:
                //删除功能
                stu1.delete_stu();
                break;
            case 6:
                //显示学生信息
                stu1.showInfo();
                break;
            case 7:
                //退出程序
                return 0;
                break;
            default:
                system("cls");
                break;
            }
        }
        else
        {    //归位cin标识符，不至于死循环
            cin.clear();
            cin.ignore();
            cout << "非法数据"<<endl;
            system("pause");
        }
    }
}
```