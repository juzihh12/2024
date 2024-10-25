---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: false
  outline:
    visible: true
  pagination:
    visible: false
---

# Jenkins Pipeline

## 一、Jenkins Pipeline介绍

Jenkins pipeline （流水线）是一套运行于jenkins上的工作流框架，将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排与可视化。它把持续提交流水线（Continuous Delivery Pipeline）的任务集成到Jenkins中。

pipeline 是jenkins2.X 最核心的特性， 帮助jenkins 实现从CI到CD与DevOps的转变。

持续提交流水线（Continuous Delivery Pipeline）会经历一个复杂的过程： 从版本控制、向用户和客户提交软件，软件的每次变更（提交代码到仓库）到软件发布（Release）。这个过程包括以一种可靠并可重复的方式构建软件，以及通过多个测试和部署阶段来开发构建好的软件（称为Build）。

总结：

1.Jenkins Pipeline是一组插件，让Jenkins可以实现持续交付管道的落地和实施。\
2.持续交付管道(CD Pipeline)是将软件从版本控制阶段到交付给用户或客户的完\
整过程的自动化表现。\
3.软件的每一次更改（提交到源代码管理系统）都要经过一个复杂的过程才能被发布。

## 二、为什么使用Jenkins Pipeline

本质上，Jenkins 是一个自动化引擎，它支持许多自动模式。 Pipeline向Jenkins中添加了一组强大的工具, 支持简单的CI到全面的CD pipeline。通过对一系列的相关任务进行建模, 用户可以利用pipeline的很多特性:

1、代码：Pipeline以代码的形式实现，使团队能够编辑，审查和迭代其CD流程。\
&#x20;   2、可持续性：Jenkins重启或者中断后都不会影响Pipeline Job。\
&#x20;   3、停顿：Pipeline可以选择停止并等待人工输入或批准，然后再继续Pipeline运行。\
&#x20;   4、多功能：Pipeline支持现实复杂的CD要求，包括循环和并行执行工作的能力。\
&#x20;   5、可扩展：Pipeline插件支持其DSL的自定义扩展以及与其他插件集成的多个选项。

\
DSL 是什么？

DSL 其实是 Domain Specific Language 的缩写，中文翻译为领域特定语言（下简称 DSL）；而与 DSL相对的就是GPL，这里的GPL并不是我们知道的开源许可证，而是 General Purpose Language的简称，即通用编程语言，也就是我们非常熟悉的 Objective-C、Java、Python 以及 C 语言等等。

## 三、Jenkins pipeline 入门

pipeline支持两种语法：

　-Declarative：声明式

　-Scripted pipeline ：脚本式

### 1、脚本式语法pipeline

声明式语法包括以下核心流程：

1.pipeline : 声明其内容为一个声明式的pipeline脚本\
2.agent:  执行节点（job运行的slave或者master节点）\
3.stages: 阶段集合，包裹所有的阶段（例如：打包，部署等各个阶段）\
4.stage:  阶段，被stages包裹，一个stages可以有多个stage\
5.steps:  步骤,为每个阶段的最小执行单元,被stage包裹\
6.post:   执行构建后的操作，根据构建结果来执行对应的操作

1、创建一个简单的pipeline

```
pipeline{
  agent any
  stages{
    stage("This is first stage"){
       steps("This is first steps"){
         echo "i am juzihh12"
       }
     }
   }
    post{
      always{
        echo "The process is ending"
      }
    }
}     
```

2、参数详解

<pre data-full-width="false"><code><strong>1、Pipeline：
</strong>作用域：应用于全局最外层，表明该脚本为声明式pipeline
是否必须：必须

2、agent：
作用域：可用在全局与stage内
agent表明此pipeline在哪个节点上执行
是否必须：是
参数：any,none, label, node,docker,dockerfile

agent any
#运行在任意的可用节点上

agent none
#全局不指定运行节点，由各自stage来决定

agent { label 'master' }
#运行在指定标签的机器上,具体标签名称由agent配置决定

agent { 
     node {
         label  'my-defined-label'
         customWorkspace 'xxxxxxx'
    } 
}
#node{ label 'master'} 和agent { label 'master' }一样，但是node可以扩展节点信息，允许额外的选项 (比如 customWorkspace )。

agent { docker 'python'  }
#使用指定的容器运行流水线
如：
agent {
    docker {
        image 'maven:3-alpine'
        label 'my-defined-label'
        args  '-v /tmp:/tmp'
    }
}
#定义此参数时，执行Pipeline或stage时会动态的在具有label 'my-defined-label'标签的node提供docker节点去执行Pipelines。 docker还可以接受一个args，直接传递给docker run调用。


agent {
    // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
    dockerfile {
        filename 'Dockerfile.build'
        dir 'build'
        label 'my-defined-label'
        additionalBuildArgs  '--build-arg version=1.0.2'
    }
}
</code></pre>

### 2、声明式语法Declarative

### 1、environment

environment指令指定一系列键值对，这些键值对将被定义为所有step或stage-specific step的环境变量，具体取决于environment指令在Pipeline中的位置。该指令支持一种特殊的方法credentials()，可以通过其在Jenkins环境中的标识符来访问预定义的凭据。对于类型为“Secret Text”的凭据，该 credentials()方法将确保指定的环境变量包含Secret Text内容；对于“标准用户名和密码”类型的凭证，指定的环境变量将被设置为username:password并且将自动定义两个附加的环境变量：MYVARNAME\_USR和MYVARNAME\_PSW。

```
pipeline{
  agent any
  environment {
    CC = 'clang'
  }
  stages {
    stage ("Example"){
      steps {
        sh 'printenv'
      }
    }
  }
}
```

### 2、options

options指令允许在Pipeline本身内配置Pipeline专用选项。Pipeline本身提供了许多选项，例如buildDiscarder，但它们也可能由插件提供，例如 timestamps。

可用选项

&#x20;       buildDiscarder：  pipeline保持构建的最大个数。用于保存Pipeline最近几次运行的数据，例如：options { buildDiscarder(logRotator(numToKeepStr: '1')) }\
　　disableConcurrentBuilds： 不允许并行执行Pipeline,可用于防止同时访问共享资源等。例如：options { disableConcurrentBuilds() }\
　　skipDefaultCheckout：跳过默认设置的代码check out。例如：options { skipDefaultCheckout() }\
　　skipStagesAfterUnstable： 一旦构建状态进入了“Unstable”状态，就跳过此stage。例如：options { skipStagesAfterUnstable() }\
　　timeout： 设置Pipeline运行的超时时间，超过超时时间，job会自动被终止，例如：options { timeout(time: 1, unit: 'HOURS') }\
　　retry：  失败后，重试整个Pipeline的次数。例如：options { retry(3) }\
　　timestamps： 预定义由Pipeline生成的所有控制台输出时间。例如：options { timestamps() }

```
pipeline {
  agent any
  options {
    timeout(time:1,unit:'HOURS')
  }
  stages {
    stage('Example'){
      steps {
        echo "Hello World"
      }
    }
  }
}
```

### 3、parameters

parameters指令提供用户在触发Pipeline时的参数列表。这些参数值通过该params对象可用于Pipeline stage中，具体用法如下：

作用域：被最外层pipeline所包裹，并且只能出现一次，参数可被全局使用

好处：使用parameters好处是能够使参数也变成code,达到pipeline as code，pipeline中设置的参数会自动在job构建的时候生成，形成参数化构建

可用参数\
　　string\
　　　　A parameter of a string type, for example: parameters { string(name: 'DEPLOY\_ENV', defaultValue: 'staging', description: '') }\
　　booleanParam\
　　　　A boolean parameter, for example: parameters { booleanParam(name: 'DEBUG\_BUILD', defaultValue: true, description: '') }\
　　目前只支持\[booleanParam, choice, credentials, file, text, password, run, string]这几种参数类型，其他高级参数化类型还需等待社区支持。

```
pipeline {
  agent any
  parameters {
    string(name:'juzihh12',defaultValue:'my name is juzihh12',description:'my name is juzihh12')
    booleanParam(name:'suansuan',defaultValue:true,description:'name')
  }
  stages {
    stage("stage1"){
      steps {
        echo "juzihh12"
        echo "suansuan"
      }
    }
  }
}
```

### 4、triggers

triggers指令定义了Pipeline自动化触发的方式。目前有三个可用的触发器：cron和pollSCM和upstream。

用域：被pipeline包裹，在符合条件下自动触发pipeline

cron\
　　接受一个cron风格的字符串来定义Pipeline触发的时间间隔，例如：

&#x20;       triggers { cron('H 4/\* 0 0 1-5') }\
pollSCM\
　　接受一个cron风格的字符串来定义Jenkins检查SCM源更改的常规间隔。如果存在新的更改，则           Pipeline将被重新触发。例如：triggers { pollSCM('H 4/\* 0 0 1-5') }

```
pipeline {
  agent any
  trggers {
    cron('H 4/* 0 0 1-5')
  }
  stages {
    stage("Example"){
      steps {
        echo "Hello World"
      }
    }
  }
}
```

### 5、tools

通过tools可自动安装工具，并放置环境变量到PATH。如果agent none，这将被忽略。

Supported Tools([Global Tool Configuration](http://192.168.1.91:8080/jenkins/configureTools))

工具名称必须在Jenkins 管理Jenkins → 全局工具配置中预配置\
　　maven\
　　jdk\
　　gradle

```
pipeline {
  agent any
  tools {
    maven 'apache-maven-3.0.1'
  }
  stages {
    stage("Example"){
      stemps {
        sh 'mvn --version'
      }
    }
  }
}
```

### 6、input

stage 的 input 指令允许你使用 input step提示输入。 在应用了 options 后，进入 stage 的 agent 或评估 when 条件前，stage 将暂停。 如果 input 被批准, stage 将会继续。 作为 input 提交的一部分的任何参数都将在环境中用于其他 stage

配置项：

message  必需的。 这将在用户提交 input 时呈现给用户。

id    input 的可选标识符， 默认为 stage 名称。

ok   \`input\`表单上的"ok" 按钮的可选文本。

submitter    可选的以逗号分隔的用户列表或允许提交 input 的外部组名。默认允许任何用户。

submitterParameter    环境变量的可选名称。如果存在，用 submitter 名称设置。

parameters     提示提交者提供的一个可选的参数列表。

```
pipeline {
  agent any
  stages {
    stage("Example"){
      imput {
        message "Should we continue?"
        ok "Yes, We should"
        submitter "juzihh12,suansuan"
        parameters {
          string(name:'PERSON',defaultValue:'juzihh12',description:'why say')
        }
      }
      steps {
        echo "Hello, $(PERSON),nice to meet you"
      }
    }
  }
}
```

### 7、when

when指令允许Pipeline根据给定的条件确定是否执行该阶段。该when指令必须至少包含一个条件。如果when指令包含多个条件，则所有子条件必须为stage执行返回true。这与子条件嵌套在一个allOf条件中相同（见下面的例子）。 更复杂的条件结构可使用嵌套条件建：not，allOf或anyOf。嵌套条件可以嵌套到任意深度。

内置条件

branch\
　　　　当正在构建的分支与给出的分支模式匹配时执行，例如：when { branch 'master' }。请注意，这仅适用于多分支Pipeline。\
　　environment\
　　　　当指定的环境变量设置为给定值时执行，例如： when { environment name: 'DEPLOY\_TO', value: 'production' }\
　　expression\
　　　　当指定的Groovy表达式求值为true时执行，例如： when { expression { return params.DEBUG\_BUILD } }\
　　not\
　　　　当嵌套条件为false时执行。必须包含一个条件。例如：when { not { branch 'master' } }\
　　allOf\
　　　　当所有嵌套条件都为真时执行。必须至少包含一个条件。例如：when { allOf { branch 'master'; environment name: 'DEPLOY\_TO', value: 'production' } }\
　　anyOf\
　　　　当至少一个嵌套条件为真时执行。必须至少包含一个条件。例如：when { anyOf { branch 'master'; branch 'staging' } }

```
pipeline {
  agent any
  stages {
    satge("Example Build"){
      steps {
        echo "Hello World"
      }
    }
    stage("Example Deploy"){
      when {
        allOf {
          branch 'production'
          environment name: 'DEPLOY_TO'
          value:'production'
        }
      }
      steps {
        echo"Deploying"
      }
    }
  }
}
```

### 8、Parallel

Declarative Pipeline近期新增了对并行嵌套stage的支持，对耗时长，相互不存在依赖的stage可以使用此方式提升运行效率。除了parallel stage，单个parallel里的多个step也可以使用并行的方式运行。

```
pipeline {
  agent any 
  satges {
    stage("Non-Parallel Stage"){
      steps {
        echo "This stage will be executed first"
      }
    }
    stage("Parallel Stage"){
      when {
        branch 'master'
      }
      parallel {
        stage('Branch A'){
          agent {
            label "for-branch-a"
          }
          steps {
            echo "On Branch A"
          }
        }
        stage('Branch B'){
          agent {
            label "for-branch-b"
          }
          steps {
            echo "On Branch B"
          }
        }
      }
    }
  }
}
```

## 四、Pipeline Scripted语法

Groovy脚本不一定适合所有使用者，因此jenkins创建了Declarative pipeline，为编写Jenkins管道提供了一种更简单、更有主见的语法。但是由于脚本化的pipeline是基于groovy的一种DSL语言，所以与Declarative pipeline相比为jenkins用户提供了更巨大的灵活性和可扩展性。

### 1、流程控制

　pipeline脚本同其它脚本语言一样，从上至下顺序执行，它的流程控制取决于Groovy表达式，如if/else条件语句，举例如下：

```
node {
    stage('Example') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

### 2、Declarative pipeline和Scripted pipeline的比较

共同点：\
　　两者都是pipeline代码的持久实现，都能够使用pipeline内置的插件或者插件提供的stage，两者都可以利用共享库扩展。\
&#x20;   区别：\
　　两者不同之处在于语法和灵活性。Declarative pipeline对用户来说，语法更严格，有固定的组织结构，更容易生成代码段，使其成为用户更理想的选择。但是Scripted pipeline更加灵活，因为Groovy本身只能对结构和语法进行限制，对于更复杂的pipeline来说，用户可以根据自己的业务进行灵活的实现和扩展。
