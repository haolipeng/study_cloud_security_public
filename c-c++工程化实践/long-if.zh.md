# 使用分支表简化if-else逻辑

## 示例场景

假设有一个程序需要处理用户输入的(a, b, c, d, e)命令并执行相应操作，传统实现如下：

```
if cmd == ‘a’
	do_something_for_a();
else if cmd == ‘b’
	do_something_for_b();
......(skip)
else
	do_error();
```

一个if-else分支处理一个命令，然后最后一个else语句是处理错误情况的。
让我们模拟下真实场景：

```c
#include <stdio.h>
#include <unistd.h>

int long_if(char cmd)
{
	int ret;

	switch (cmd) {
	case 'a':
		printf("command is a\n");
		/* code for command a */
		ret = 1;
		break;
	case 'b':
		printf("command is b\n");
		/* code for command a */
		ret = 2;
		break;
	case 'c':
		printf("command is c\n");
		/* code for command a */
		ret = 3;
		break;
	case 'd':
		printf("command is d\n");
		/* code for command a */
		ret = 4;
		break;
	case 'e':
		printf("command is e\n");
		/* code for command a */
		ret = 5;
		break;
    //后续这里还会添加很多其他的命令
	default:
		printf("Unidentified command\n");
		/* code for error */
		ret = -1;
		break;
	}
	return ret;
}


int main(int argc, char *argv[])
{
	char cmd;

	if (argc != 2) {
		printf("usage: ./a.out command(a|b|c|d|e)\n");
		return 1;
	}

	printf("result=%d\n", long_if((char)argv[1][0]));
	return 0;
}
```

这个 long-if 函数是用来处理用户输入的命令的。很多有经验的程序员喜欢用 switch-case 来写这种逻辑，但其实它和 if-else 一样，都是一个个判断命令该执行什么操作。

代码长一点倒不是大问题，问题是程序以后可能要经常改动：

- 可能要增加新命令
- 命令的格式可能会变（比如从字母a/b/c改成数字1/2/3）
- 业务逻辑可能要调整，比如每个命令的处理函数

每次修改都要把整个 if-else 或者 switch-case 看一遍，找到需要改的地方。有时候要改很多处，有时候还要加新的判断。

最麻烦的是，随着程序功能越来越多，这些判断会变得越来越长。等有几十个命令的时候，改起来就特别容易出错，而且很花时间。

所以我们需要更好的写法...



现在我要介绍一种"**数据和逻辑分离**"的技巧：

- **数据**：指那些经常变化的外部输入（比如用户命令）
- **逻辑**：处理数据的程序代码（通常比较固定）

具体来说：

- **数据部分**：用户的各种命令
- **逻辑部分**：处理这些命令的程序代码

怎么实现分离呢？我们可以这样组织数据：

- (a, 处理函数_a)
- (b, 处理函数_b)
- (c, 处理函数_c)
- (d, 处理函数_d)
- (e, 处理函数_e)

这样就建立了一个"命令-处理函数"的对应表。最后我们只需要写一个通用的处理逻辑：

```
for each pair in the list
	if user's command == command in the pair
		call handler in the pair
```

这段核心代码几乎永远不需要修改。
就算以后要新增命令、删除命令，或者调整功能需求，
上面的处理逻辑都完全不用动。

下面我用具体代码来演示如何实现这种分离：

```c
static int handler_a(char cmd)
{
	printf("command is a\n");
	return cmd - 'a' + 1;
};

static int handler_b(char cmd)
{
	printf("command is b\n");
	return cmd - 'a' + 1;
};

static int handler_c(char cmd)
{
	printf("command is c\n");
	return cmd - 'a' + 1;
};

static int handler_d(char cmd)
{
	printf("command is d\n");
	return cmd - 'a' + 1;
};

static int handler_e(char cmd)
{
	printf("command is e\n");
	return cmd - 'a' + 1;
};

//命令行处理器
struct cmd_handler {
	char cmd;//具体命令
	int (*handler)(char);//命令处理函数
};

int short_if(char cmd)
{
	int i;
	int ret = -1;
    //定义所有命令的合集
	struct cmd_handler chandlers[] = {
		{'a', handler_a},
		{'b', handler_b},
		{'c', handler_c},
		{'d', handler_d},
		{'e', handler_e}
    };
	
	for (i = 0; i < sizeof(chandlers)/sizeof(chandlers[0]); i++) {
        //遍历每个命令并调用其处理函数
		if (chandlers[i].cmd == cmd) {
			ret = chandlers[i].handler(chandlers[i].cmd);
			break;
		}
	}

	if (ret < 0)
		printf("Unidentified command\n");
	return ret;
}
```

我把原来的 long_if 替换成了 short_if。这个 short_if 最大的特点是：把会变动的部分都整理成了清晰的数据结构。

具体是怎么做的呢？

1. 定义了一个结构体 struct cmd_handler，用来存：
   - 命令是什么（比如字母a/b/c）
   - 对应的处理函数是什么
2. 然后把所有命令和处理函数的对应关系，用一个数组存起来：

```
struct cmd_handler chandlers[] = {
		{'a', handler_a},
		{'b', handler_b},
		{'c', handler_c},
		{'d', handler_d},
		{'e', handler_e}
    };
```

这样做有什么好处？

➕ 要加新命令？只需要在数组里加一行对应关系

✂️ 要删命令？直接删掉数组里对应的那行

🔄 要改命令？也只需要修改数组里的对应关系

不管怎么改，都只需要动这个数组，核心的处理逻辑完全不用碰。



可能有些小伙伴会觉得："这样写代码不是更复杂了吗？"

但我们要记住：
1️⃣ 程序是要长期维护的
2️⃣ 功能只会越来越多

👉 现在需求可能是处理a/b/c/d/e这几个命令
👉 下个月可能要加个f命令
👉 明年说不定要支持1/2/3/4/5这些数字命令

如果用老方法if-else：

- 每次修改都要在几百行代码里找对应的判断条件
- 加一个命令就要新增一个if分支
- 时间久了代码会变成超长的"面条代码"

用了数据分离的新方法后：

- ✨ 改需求时只需要关注数据部分（那个对应关系数组）
- ✨ 核心代码完全不用动
- ➕ 加命令？在数组里加一行就行
- ➖ 删命令？直接删掉数组里那行
- 📂 处理函数太大？可以拆到单独文件里
- ❌ 甚至忘记删掉没用的处理函数也没关系（不影响运行）
