---
layout: post
title:  "现代C++错误码之经验谈"
date:   2020-08-18
---

<p>近几年接触了一些C++大型项目，其中不乏有很优秀的代码，但也有不少代码在实现的时候缺少极致深度思考，导致代码的安全性，可读性和复杂性都有很大的影响。今天和大家讨论一下一个大型系统中，返回错误码的问题。</p>


## 一 问题
不少大型项目为了给用户提供易用的接口，所以设计的一套API都是以返回固定的error code(有的系统也叫status code)作为返回值，例如:

```c++
// error_code.h
// 需要注意的是，这个错误码是整个系统都使用这一个类型。
enum ErrorCode {
    // 更多的OK_XXX错误码
    OK_XXX = 103;
    OK_RPC_FAILED = 102;
    OK_KYEY_NOT_FOUND = 101;
    
    OK = 0;
    
    // 网络相关错误码
    NETWORKING_FAILURE = -101,
    HOST_ERROR = -102,
    DNS_ERROR = -103,

    // 机器负载相关错误码
    DISK_ERROR = -201;
    MEMORY_FULL = -202;
    XXXX = -203;
    XXXX = -204;
    
    // 然后后面可能还有几百条和业务相关的错误码
}
```

然后要求整个项目的绝大多数方法都返回这个错误码，以便于函数调用过程中一层层往上直接返回。假设我们写了一个简单的util方法split_addr()，这个方法要求我们将一个ip:port格式的字符串split出ip和port两部分。则按照这种错误码要求，我们需要这样实现：

```c++
ErrorCode split_addr(const std::string &origin_str, std::pair<std::string, int16_t> *result) {
    if (result == nullptr || !origin_str.contains(":")) {
        return ErrorCode::SPLIT_INVALID_ARGUMENT;
    }

    const auto split_index = origin_str.find(":");
    if (split_index out of range) {return ErrorCode::SPLIT_ERROR;}
    result->first = origin_str.substr(0, split_index);
    // 你还可以加一些其他的trim或者bound的检查等
    try {
        result->second = static_cast<>(std::stoi(result->second));
    } catch (const exception &e) {
        // log this error
        return ErrorCode::SPLIT_ERROR;
    }
    return ErrorCode::OK;
}

// 测试
std::pair<std::string, int16_t> ip_and_port;
split_addr("127.0.0.1:8888", &ip_and_port);
cout << ip_and_port.first; // 输出127.0.0.1
cout << ip_and_port.second; // 输出8888
```

到这里，你以为写完就可以跑了吗？当然不是，你还得将你新定义的这几个错误码加入到ErrorCode的枚举里去，怎么加？首先你得浏览前面我们提到的已定义的几百个错误码中是否有你需要的错误码，如果没有，那么很庆幸，你直接将这几个加入到最后:

```c++
enum ErrorCode {
    // 更多的OK_XXX错误码
    OK_XXX = 103;
    OK_RPC_FAILED = 102;
    OK_KYEY_NOT_FOUND = 101;
    
    OK = 0;
    
    // 网络相关错误码
    NETWORKING_FAILURE = -101,
    HOST_ERROR = -102,
    DNS_ERROR = -103,

    // 机器负载相关错误码
    DISK_ERROR = -201;
    MEMORY_FULL = -202;
    XXXX = -203;
    XXXX = -204;
    
    // 然后后面可能还有几百条和业务相关的错误码

    // 加到了这里
    SPLIT_ERROR = 701,
    SPLIT_INVALID_ARGUMENT = 702,
}
```

OK，大功告成，你的代码成功的跑起来了！

接着我们来回顾刚才的开发过程，想想其中存在哪些不安全行为。在开发这样一个小小的split helper的过程中你会经历好几次危险的过程，稍有不慎，你将你的代码置于危险之地并在线上环境跑起来。
简单列举一下其中的一些可能存在问题的地方:
1. 在将我们新增的错误码添加到ErrorCode里的过程，我们需要仔细检查其中是否有可能适合我们的错误码。这个过程的问题在于有些错误码的定义你可能知道其不能代表我新增的错误码，但是有些你不太好区分(虽然我们可以按类别将同一类型的错误码放在一块，但有些毕竟不相同)，例如INVALID_ARGUMENT，NULLPTR_ARGUMENT等等各式各样的错误码定义，因为这些错误码都是不同程序员写的，很难揣摩其他人的意图。简而言之就是你并不知道（或者很难确定）现有的代码里的那些错误码是否合适你。
2. 在此之后，模块B的同学可能需要在模块B中新加一个split的方法用于split其自己的一个以下划线为分隔符的字符串。那么这个同学的开发过程，可能会变得较为艰苦，因为方法里有一些SPLIT相关的错误码，他需要仔细甄别这些错误码能否拿来用，也许有时候有部分可能拿来。（如果不太理解这一点的话，看完下面的例子就清楚。）
3. ErrorCode这个错误码本来是给用户提供的接口所使用的错误码，其更改实际上是对用户的语义也造成了更改。即使你用注释告诉用户只能用OK和不OK两个状态，但是在编译器角度，你没办法区别开，这就是所谓的编译安全性。用户或者我们自己的其他开发者如果一不小心写出了一个这样的代码，在编译器角度是安全的，可以编译通过，但很显然这样的代码写出来是会对线上造成巨大隐患：

```c++
// error.h
// 注意，这里是我们全局唯一的一个ErrorCode类型，其包含系统所以错误码
enum ErrorCode {
    // 这里已经有四百个错误码枚举值
}
```
接着，A同学开发了split_x()方法，按照上述开发逻辑，我们需要做如下事情：
(1) 增加A模块的split_x方法所需要的错误码到全局的error.h中
```c++
// error.h
// 注意，这里是我们全局唯一的一个ErrorCode类型，其包含系统所以错误码
enum ErrorCode {
    // 这里已经有四百个错误码枚举值

    // A同学在模块A中给split_x增加了这个错误码
    ERROR_SPLIT_INVALID_ARGUMENT = 701;
}
```
(2) A同学开发split_x方法
```c++
// A模块中
ErrorCode spilit_x(xxx, yyy) {
    if (do sth) {
        return ErrorCode::ERROR_SPLIT_INVALID_ARGUMENT;
    } 
    // 也许还有可能返回其他几个错误码，但不重要
    return ErrorCode::OK;
}
```

由于大家局部的状态码都是往error.h中添加，那么数个月后，`error.h`可能又增加了10个错误码，可能长成这样子了:
```c++
// error.h
// 注意，这里是我们全局唯一的一个ErrorCode类型，其包含系统所以错误码
enum ErrorCode {
    // 这里已经有四百个错误码枚举值

    // A同学在模块A中给split_x增加了这个错误码
    ERROR_SPLIT_INVALID_ARGUMENT = 701;
    
    // 这里又有了10个错误码
}
```

然后，B同学在B模块需要开发一个他自己的helper method，也是和split相关的，但不完全一样，那么他的开发过程也和A同学类似:
```c++
// B模块中
ErrorCode split_y(xxx) {
    if (xxx != nullptr) {
        return ErrorCode::NULLPTR_ERROR;
    }

    return ErrorCode::OK;
}
```
接着B同学需要做2件事情：
第一，查阅`error.h`文件，确认`split_y()`方法中返回的错误码是否已在`error.h`中定义，如果全部都定义了，则不再新增新的错误码。那么这个过程就是上面第2点提到的，B同学则需要小心甄别这些错误码，因为他不知道A同学给一些错误码的命名方式是和自己的思维一样，例如B需要一个`ERROR_SPLIT_NULLPTR`的错误码，但是错误码中没有`ERROR_SPLIT_NULLPTR`，不过却有`ERROR_SPLIT_INVALID_ARGUMENT`，接着B同学需要去查找这个错误码使用的地方，即去查看A同学写的split_x方法如何用这个错误码的，是否可以用`ERROR_SPLIT_INVALID_ARGUMENT`来表示自己的参数是个nullptr的情况。说到这里，我们为B同学感到丝丝的担忧，因为他太难了。其实对B同学而言，开发B模块的split_y，本没有职责去，也不应该去看A模块中的`split_x`方法的具体实现。
第二， 他通过自己仔细的甄别，认为还是需要加一个`ERROR_SPLIT_NULLPTR`的错误码，反正到这里，对B而言终于解脱了:
```c++
// error.h
// 注意，这里是我们全局唯一的一个ErrorCode类型，其包含系统所以错误码
enum ErrorCode {
    // 这里已经有四百个错误码枚举值

    // A同学在模块A中给split_x增加了这个错误码
    ERROR_SPLIT_INVALID_ARGUMENT = 701;
    
    // 这里又有了10个错误码
     
    ERROR_SPLIT_NULLPTR = 801;     // B同学新增的错误码，也可能他还增加了其他错误码
}
```

到这里，大家还记得代码库里长什么样吗？我简单描述一下，`error.h`有几百个错误码，其中有个`ERROR_SPLIT_INVALID_ARGUMENT`是A同学给`split_x()`方法返回用的，还有个`ERROR_SPLIT_NULLPTR`是B同学给B模块中的`split_y()`用的。嗯，接着C同学可能在某时刻B模块中使用`split_y()`方法，但是C同学真的不够聪明，写出了如下代码:
```c++
// 代码片段c
void f() {
    // do buiness

    if (ErrorCode::ERROR_SPLIT_INVALID_ARGUMENT != split_y(result)) {
        // 处理错误
    } else {
        // 正常流程，做一些事情, 使用result
    }
}
```

如果result在经历九九八十一难之后到达这里成为一个nullptr之后，可怕的后果即将发生，轻则在else处使用result，导致程序core dump，重则esle处没有使用result，而是把result再经过山路十八弯传到你都不知道传到的那个地方去了，导致最终线上出现问题，你都很难排查得出最原始的出错位置。这里出现问题的原因是在于`split_y`对空指针返回的是`ERROR_SPLIT_NULLPTR`，而不是`ERROR_SPLIT_INVALID_ARGUMENT`,所以会走进else分支而出问题。

为什么这种情况很难避免呢？因为我们要想避免C同学的这部分代码合入到你代码库中，只有2种方法，**单元测试和code review**。 而C同学的这段错误代码是很难被这两种方式检查出来的，为什么呢？首先说单元测试，业务逻辑往往很复杂，C++也不是一个内存安全的语言，因此在`if()`之前你很难知道你的unit test怎么去构造才会使得result为空。因为f是业务逻辑，不是单独的一个方法，C同学的单测只能覆盖到这些业务逻辑层的各种case，这里的case很显然不容易覆盖到。其次对于`split_y()`的测试，是B同学写的`split_y()`的单元测试，B同学写的这个测试很显然能够正确运行，因为他一定只有返回了ERROR_SPLIT_NULLPTR他的测试才能通过，但这对于C同学而言一无所知。再来说code review，对于reviewer而言，看到这里代码，他如果不非常熟悉error code里的几百个枚举值分别表示什么，他就很难catch住这个if()的错误。并且reviewer在看到这个if()的第一直觉也就是下意识的认为很棒，这个if()做了一次错误码判断。（主要原因还是在于，reviewer只能检查代码的逻辑正确和结构正确，实际运行起来的细节根本cover不住。）嗯，故事差不多就是这个样子。

很显然，造成上述问题的原因可能是多种的，what ever，作为一门极度危险语言C++的开发者(这里的危险是指你要处处小心)，从根源上杜绝不安全行为是基本准则。

在介绍如何使用安全性高的返回值之前，我再次强调一下编译安全这个概念。我们不能假定任何其他开发者(其实也包括我们自己)都有着无穷的智慧可以小心翼翼地处理任何细节，而是要假定其他开发者都是一个“大笨蛋”，甚至说其他开发者就是属于“杠精”，专门去写出不安全的代码使得程序出现问题，并且要视这种情况为普通发生的情况。那么在对付这样情况的时候，我们就需要使用现代化C++更加安全的技术或者方法论使得“杠精”无处遁形，即写出任何一种可能到处问题的代码的时候，都能在编译阶段让“杠精”编译不通过，这就是所谓的编译安全(题外话:C++20的concepts提案就是专门针对编译期安全性校验和human-readable的编译期报错的一个提案)。


## 二 原则
现代化C++错误码或者说状态码其实并不涉及Moderen C++的任何新技巧，而只是思想上，战略上，设计上去做好这件事情。
- 原则1:更改跟函数签名有关的任何内容都视为一种API的改动（维护stable API）
- 原则1.5: “设计”不依赖“实现”，“实现”可以迭代“设计”
- 原则2: 模块间解耦，确定模块间依赖关系以及模块间的交互语义(即不要一个error code走遍天下)
- 原则3: 缩小error code的scope，不断迭代各种不同类型的error code
- 原则4: 最重要的原则:视其他人都是杠精
- 原则5: 不属于用户感知的错误码，不应当作为API的错误码一部分
- 原则6: 使用enum class instead of raw enum

## 三 简单改进
基于上述原则，我们可以简单实现一套更安全的返回码机制:

```c++
error.h
// 抽象用户API，提供一套稳定且安全的API
// 注意这个返回值是用户代码的返回error code，它应该是需要和API一样的stable, 并不应该随意更改
// 因为基于原则1, ErrorCode是API方法签名的返回值，它的任何更改都属于API级别的更改，因此在能兼容的情况下
// 应该是需要通过小版本迭代才能更改，如果新增的一个error code值不能兼容，则是需要通过大版本迭代。
enum class ErroCode uint16_t {
    OK = 0,
    X_NOT_FOUND = 1,
    B_IS_FULL = 2,
    XXXX
};

class Error {
    // some useful methods.
    static Error ok() {return Error(ErrorCode::OK, xxx);}
    static Error x_not_found() {return xxxx;}
    // ...
    bool is_ok() {xxxx}
    // ...
private:
  ErrorCode code;
  std::string message;
  // 这里可以再加一个detail代理以便于将子模块的error code作为error的一部分来扩展开
  // 这部分后续再单独讲解，这里我们主要讲解error code的安全性问题。
};
```

其次在不同的模块间，通过依赖关系，先确定模块依赖和交互关系，再基于此考虑如何定义模块间的错误码(也可以不需要错误码)。这部分由于和业务有着一定的关系，因此我不太好用代码表述出来，但不管怎样，基本的原则就是先定义模块间的依赖，再定义错误码，并且尽可能不要复用API的错误码，除非完全一样。

其次，再来说下如果我们A， B两位同学实现各自不同的split的同时，他们应该怎么做。这里的做法就可以是千奇百怪，但是要注意一点，就是
split返回的错误码绝不能往API的error code中添加。A同学可以直接返回一个inner error code:

```c++
enum class ModuleAXXXErrorCode {
    SPLIT_INVALID_ARGUMENT = 0,
    xxxxx,
    xxxxx,
};

ModuleAXXXError split() {
    // ......
}
```

B同学也可以在他的模块中实现一个他自己需要的split helper, 并返回一个自己的inner error code或者返回其他任何他自己认为OK的方式，这个例子里B同学就是返回一个bool来表示split是否成功，这种写法也是完全OK的，并且是编译安全的。至于如果你想更加细粒度的了解split内部到底出现什么错误的时候，那你可以按照自己的粒度去定义它的返回值了，但从一般经验上来讲, bool完全足够。

```c++
bool split() {
    return true or false;
}
```

需要了解enum class的可以参考（其实enum class目前而言对解决我们遇到的问题帮助不大）：
https://hownot2code.com/2016/06/14/start-using-enum-class-in-your-code-if-possible/
https://en.cppreference.com/w/cpp/language/enum


此外还有一点需要注意的是，我们不可能对任何一个方法都定义一个错误码，这是不应该的，也是不可能的，因此我们也需要掌握模块间的错误码定义，当你认为这是一个非常inner的方法时，至少你不应当扩大其错误码的scope(原则3)。其次，你还需要利用原则1.5和原则2去通过不断迭代，来提升你模块间的错误码的质量，以避免使用更大scope的错误码行为。
此外，我们也在致力于更好的解决这个问题，但由于方案的不够通用性，可能无法在C++语法上得到支持，但我们正在努力尝试去推动这样一个提案给C++标准委员会，虽然大概率不能被accept，因为这个提案是和core language相关，而不是和library相关的。此外，我们还在考虑另外一种不需要进入C++标准的方案，即开发一个hint插件，但这只是一种静态检查，也无法做到非常高的编译安全性，但hint至少能稍微改善这种情况。

## 四 总结
编写编译安全的C++的代码，是现代化C++开发者的基本素养，我们不仅要认识到不安全的代码究竟是为何不安全的，还需要从一点点的细节上去让我们的代码安全性变得更高。这里需要大家掌握的不是技能，而是对待现代化C++大型系统的开发原则和一丝不苟的开发态度。

后续的文章，我们再来详细叙述上述几种原则的原理性。

