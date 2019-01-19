
#通过自定义gradle插件来实现基于字节码插桩的AOP功能

##背景
大家都知道android studio一直以来都是依靠gradle来进行项目构建。那么在构建的过程当中是否提供了aop的切面，来暴露给我们使用呢？答案是肯定的。我们可以通过gradle配合，来完成自己的biheaver自定义插件。
那么又有人会问，如果真的要使用·aop的切面功能，直接使用诸如biheaver这样的第三方库不就可以了么？在大部分情况下确实可以满足需求，但是它们的使用场景还是有很大的局限性的。那就是，我们需要在添加apply plugin: 'com.android.application'的build.gradle当中才可以使用。也就是说，诸如biheaver这样的第三方库，它们的使用场景都是app当中，这显然是不符合一些sdk商家需求的，所以我们需要自己动手来解决这个问题。


##AOP功能实现基础
###关于gradle
首先，我们此次既然是基于自定义gradle插件实现的AOP功能，我们就用groovy语言来实现这一功能。而编写gradle插件主要有两种。
第一种：直接在build.gradle中进行编写，这样做显然是只能在自己的项目中使用的，因此我们抛弃这种方式。
第二种：就是我们今天主要来描述的方式，就是在一个独立的Module中去实现这一功能。这样，我们可以将其编译成maven库，或者jar包，来提供给第三方进行灵活使用。

###关于字节码插桩
在android编译的过程当中，都会生成相应字节码，而我们就是通过修改其字节码完成插桩功能。为此，我们需要一些帮助。本文中使用ASM框架来帮助我们进行字节码插桩。ASM框架要求我们掌握一定的字节码知识和技巧，本文会在后面描述部分的功能实现。想要知道更多的信息，我也会贴出ASM4的pdf文档，供大家参考使用。

##自定义gradle插件的实现
###创建Gralde插件的基本过程

####1.	Gradle插件使用的开发工具。
首先，我们所使用的工具就是android studio。as本身是基于IntelliJ IDEA的基础上发展而来的，再加上我们通常在android项目中使用自定义gradle，必然伴随一些android原生module的代码注入，所以使用这个工具是比较好的选择。


####2.	Gradle插件Module的创建过程。
这里我们直接创建一个android module，因为as中并没有直接提供给我们一个创建gradle插件的方法。但是并没有什么关系，我们只需要删除用不着的目录，再修改build.gradle就可以了。
在创建完module之后，我们需要删除大部分的文件夹，就剩一个build.gradle，然后创建groovy文件夹用来存放groovy代码，再创建一个resources文件夹。这个文件夹是什么用处，稍后会有详细介绍。
在经过上述处理之后，我们的工程已经变成了如下图所示：
 ![1.png](E:/222/1.png "")（请忽略.gitignore文件）
 
####3.	resources文件夹
在完成以上步骤之后，我们需要在resources文件夹下创建一个几个新的文件夹和文件，最后的目录如下图所示:

![2.png](E:/222/2.png "")
 
这个properties类型的文件，就是我们gradle插件的配置文件。它指定了我们gradle插件的入口groovy类。这里面，其实我们只需要配置一句话即可，就像这样：
implementation-class=com.lian.plugin.xxxPlugin


####4.	build.gradle文件
接下来，我们看一下我们的build.gradle文件，这个文件主要有两个功能，一个是类似于我们android工程的build.gradle文件的功能，指定各种依赖库；另一个，我们需要打包我们的gradle插件，提供给android项目使用。
讲到第一部分，我们直接在下面贴上这部分的代码：

   ```objc 
apply plugin: 'maven'
apply plugin: 'groovy'

compileGroovy {
    sourceCompatibility = 1.7
    targetCompatibility = 1.7
    options.encoding = "UTF-8"
}

repositories {
    jcenter()
    mavenCentral()
}

dependencies {
    compile gradleApi()
    compile localGroovy()
    compile 'com.android.tools.build:gradle:2.1.3'
    compile 'org.ow2.asm:asm:5.0.3'
    compile 'org.ow2.asm:asm-commons:5.0.3'
}
```
这一部分主要就是添加了我们的依赖库，其中就包含了gradleApi，LocalGroovy，gradle插件和asm库这些必要元素。

接下来，我们谈谈第二部分，这一部分，其实作用就是打包，我们也直接贴上这部分的代码：
   ```objc 
group = 'com.lianapm.plugin'
version = '1.0.1'

uploadArchives {
    version = version + '-SNAPSHOT'//if you are testing the demo I provide locally, you can uncomment this.
    repositories {
        mavenDeployer {
            snapshotRepository(url: uri('../snapshotRepo'))
        }
    }
}
```
在加上如上的代码之后，我们可以看到，在android studio右边的gradle插件选项中，就已经可以找得到我们刚写的内容了：
![3.png](E:/222/3.png "")
 
双击这个按钮，我们的gradle就会编译成一个maven库。里面就包含了我们需要的jar文件。这样，这个插件就可以使用了。

##Gradle插件的具体实现过程
###讲完了build.gradle的配置，我们就要上正题了。

####1.	Plugin入口类
我们在上文中已经指定了我们的gradle入口类，就是xxxPlugin，这里，我们在groovy文件夹下创建这个grovvy类。然后让这个类实现Plugin<Project>接口，即implements Plugin<Project>。这个接口下只有一个方法，就是void apply(T var1);，复写这个方法，这个方法就是我们AOP的主入口。在这个方法下，我们获取到了Project类型的变量。

接下来，我们就要了解一下我们在获取到这个project的变量之后，如何获取到AOP的切面了，我们在这里用到了android暴露给我们的transform接口，这个接口会在android项目编译的时候，暴露给我们该项目所有编译过程中产生的字节码，我们可以通过修改这个字节码，从而达到插桩的目的。

在介绍完上述入口之后，我们接着上文来说一下如何实现，直接贴上void apply()方法下的第一句代码：
def android = project.extensions.getByType(AppExtension)
这句话也就是为什么我在文章开头提到，诸如biheaver这种插件，必须在apply plugin: 'com.android.application'类型下的build.gradle才可以使用了。因为AppExtension只有在当build.gradle声明为apply plugin: 'com.android.application'的时候才存在。通过上面这句话，我们获取到了类型为BaseExtension的变量“android”，我们通过这个变量，来注入我们自己的transform实现类。说到这里，我们直接创建一个新的groovy类，名字就叫NewTransform好了。我们接下来实例化这个类，NewTransform transform = new NewTransform ()，再通过android.registerTransform(transform)来注入我们这个实现类。

####2.	我们来看看Transform的具体内容。
在我们新建的NewTransform中，继承自Transform的抽象类。这里，我们就一个个看一下，需要我们关注的几个方法。

首先是getName()，这个方法顾名思义，我们只需要在里面返回一个String类型的名称就可以了

然后是getInputTypes()，这个方法其实就是让我们指定我们有哪些文件类型，是需要我们拿来过滤筛选的。这里我们直接使用

   ```objc 
return ImmutableSet.<QualifiedContent.ContentType> of(CLASSES, RESOURCES, ExtendedContentType.NATIVE_LIBS)
 ```
代表了所有jar包，文件夹中，aar包中的.class文件和标准的java源文件，我们都进行筛选。

接下来是getScopes()，这个代表了我们插件的作用域。我们返回：
return TransformManager.SCOPE_FULL_PROJECT
代表作用在整个项目中。

isIncremental()，这个则代表了是否支持增量编译，我们这里选return false

最后也是最关键的一个

   ```objc 
void transform(
            @NonNull Context context,
            @NonNull Collection<TransformInput> inputs,
            @NonNull Collection<TransformInput> referencedInputs,
            @Nullable TransformOutputProvider outputProvider,
            boolean isIncremental)
            
 ```
这个inputs就是我们transform接口下暴露的字节码。我们通过这个变量，就可以拿到在编译过程中所产生的所有字节码并在其中加上我们的“私货”。具体怎么做呢，我们需要对这个inputs变量进行遍历，这块遍历分为了两部分内容，第一部分是遍历jar，第二部分是遍历目录，下面贴上代码：

   ```objc 
/**遍历输入文件*/
        inputs.each { TransformInput input ->
            /**
             * 遍历jar
             */
                input.jarInputs.each{
 				...
}
/**
                     * 遍历目录
                     */
                       input.directoryInputs.each{
                        ...
}
				}
				  
```
具体其中的实现代码，我这里就不加赘述。此文主要还是描述整个实现流程。

##ASM库的使用过程
###1.	讲到这里，我们的ASM库就要发挥它的作用了。
以上的内容中，我们已经在inputs中拿到了class文件的字节码，接下来我们需要ASM库的帮助来完成接下来的工作。

首先，我们大概了解一下ASM库，它给我们提供了ClassReader，ClassWriter，ClassVisitor，MethodVisitor这些关键类。ClassReader顾名思义，就是class的读取类，ClassWriter是class的改写类，ClassVisitor是class的筛选类，而MethodVisitor则是class中方法的筛选类。除此之外，还有一个关键类，AdviceAdapter，这个类本身是一个抽象类，继承了GeneratorAdapter类，实现了Opcodes接口。

从Opcodes说起，Opcodes接口类本身的作用，是为了方便我们使用字节码，对字节码的一个封装，我们点进去可以看到一系列二百多个int类型变量，里面每一个变量就代表了一个字节码指令，在我们不熟悉的时候，可以依靠这个接口类来获取我们需要的字节码指令。其中，最顶端的两行
   ```objc 
int ASM4 = 4 << 16 | 0 << 8 | 0;
    int ASM5 = 5 << 16 | 0 << 8 | 0;
```
则代表了ASM指令的两个主要版本，针对这两个版本在实现细节上会有区别，本文使用了ASM4的指令版本。

接下来是GeneratorAdapter类，这个类往下看，继承自LocalVariablesSorter，而LocalVariablesSorter正是继承自MethodVisitor抽象类，也就是说GeneratorAdapter和LocalVariablesSorter是MethodVisitor部分功能实现的一个封装，而AdviceAdapter则整合了GeneratorAdapter和Opcodes，来提供给我们使用。

在上面介绍了一下ASM基本的类和功能之后，我们回到项目中，我们通过上面的inputs获取到的byte[] srcClass的字节数组，这个就是我们项目的字节码。我们新建一个XXClassVisitor类，继承ClassVisitor，然后通过以下代码，把我们新建的ClassVisitor改写的代码注入到原来的字节码当中。
  ```objc 
        ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS)
        ClassVisitor adapter = new XXClassVisitor(classWriter)
        ClassReader cr = new ClassReader(srcClass)
            cr.accept(adapter, ClassReader.EXPAND_FRAMES) 
return classWriter.toByteArray()
```
这个classWriter.toByteArray()就是我们改写之后的字节码。

###2.	接下来我们看一下	XXClassVisitor，它需要复写以下几个方法：
  ```objc 
void visit(int version, int access, String name, String signature, String superName, String[] interfaces)
void visitInnerClass(String name, String outerName, String innerName, int access)
MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions)
void visitEnd()
```
我来一一说一下上述方法的作用。void visit就是这个class类的入口方法，我们知道上面我们拿到了所有的class字节码，通过上面的方法，他们会一个一个的进入到这个方法中。我们这个visit方法要做的，就是筛选我们需要修改的class。上面几个参数，分别是如下的含义：
  ```objc
int version	版本号

int access	访问标志

String name	类的全限定名称

String signature	泛型签名

String superName	父类

String[] interfaces	接口
```

这样就很明显了，我们需要筛选class的全称，或者父类，或者实现的接口。例如，我们需要排除所有安卓的系统类，那么我们只要把name.startsWith('android')全部剔除就可以了。

接下来visitInnerClass，这个就顾名思义了，就是class的内部类，本文中并没有用到，这里我就不展开描述了。

下面是MethodVisitor visitMethod，这个是一个关键方法。它会暴露当前class下所有的方法。我们在visit中筛选出当前的类是我们需要修改的类之后，接下来会在这个方法中，暴露这个类下所有方法的字节码。我们对比一下这个方法下的参数和visit的区别，可以看到它有两个新增，就是String desc和String[] exceptions。第一个String desc 这个变量代表的是当前过滤的字节码类的参数。另一个是String[] exceptions其实就是这个类抛得异常类的名称。我们这里暂时用不到这个参数。visitMethod这个方法具体如何使用，我们在下面再来详细讲解。

最后一个void visitEnd()就是结束遍历该类的方法。

###3.	MethodVisitor如何替换指定类下的方法
我们在上述的visitMethod(int access, String name, String desc, String signature, String[] exceptions)方法中已经·获取到了当前类下面方法的参数信息。但是我们看到，并没有直接给我们方法的字节码，而返回类型是一个MethodVisitor，也就是说，返回的是一个方法的全部字节码。那么我们该如何获取当前方法的字节码呢，看下述代码：
  ```objc
MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions)
 ```
这个cv，就是ClassVisitor类型的全局变量。他是ClassVisitor类下的一个全局变量，我们在上述XXClassVisitor继承ClassVisitor类的时候，就已经将当前暴露给我们的类的字节码本身赋值给这个cv变量了。我们通过这个类下面方法的具体描述，就能确定目前所观察的是哪个方法，就能获取到这个方法的字节码。

我们可以看到上面这个methodVisitor的局部变量其实就是没有修改过的method字节码。我们既然要修改字节码，就要新建一个XXMethodVisitor的类，继承AdviceAdapter。这里并没有直接继承MethodVisitor，因为其中很多的操作，我们没有必要亲自去完成，ASM库已经帮我们封装了一层。我们这个XXMethodVisitor的主要作用是进行初始化，我们在他的构造函数里给父类super(Opcodes.ASM4, mv, access, name, desc)添加Opcodes.ASM4的标识。并且，我们可以Override所有AdviceAdapter的方法，进行打印或者相关的操作，再添加一个super方法，来全局注入这些打印，相当于一个代理类。因为，我们在接下来使用过程中，并不会去复写AdviceAdapter下所有的方法，我们只去复写我们所需要用到的。

接下来，我们就可以进行具体的操作了。首先，我们创建一个自己的MethodVisitor对象。MethodVisitor adapter = null。当我们遇到了一个需要修改的方法，例如，我们最常见的onClick，我们应该怎么描述呢？两个条件：第一，name，也就是方法名称的String为“onClick”，第二，interfaces，实现的接口当中必须包含OnClickListener。当然，你也可以加第三个，他的参数中包含一个View类型的变量。我们怎么描述OnClickListener这个接口呢，直接贴上代码吧“android/view/View$OnClickListener”，通过结构我们可以看到，android/view/View为这个接口所在的class类名，$后面则是这个接口的名称。

我们在判断获取到这个方法之后，我们就可以通过上面的adapter空对象来进行操作了，这里直接贴上代码：
  ```objc
  adapter = new XXMethodVisitor(methodVisitor, access, name, desc) {
                @Override
                protected void onMethodEnter() {
                    super.onMethodEnter()
                    methodVisitor.visitVarInsn(Opcodes.ALOAD, 1)

                    methodVisitor.visitMethodInsn(Opcodes.INVOKESTATIC, "com/xxx/xxx/HelperProxy", "onClick", "(Landroid/view/View;)V", false)
                }
            }
            
 ```
这个onMethodEnter就是我们最常用的方法，他的含义就是在这个方法最前端进入的时候的回调，用来添加方法最前端的代码。而另一个onMethodExit则是在这个方法的最末端即将退出这个方法时的回调，用来添加方法最末端的代码。

methodVisitor.visitVarInsn(Opcodes.ALOAD, 1)这句话，则代表了获取onClick(View view)里view这个局部变量的值。而接下来的一句话：methodVisitor.visitMethodInsn(Opcodes.INVOKESTATIC, "com/xxx/xxx/HelperProxy", "onClick", "(Landroid/view/View;)V", false)则是新建一个static类型的变量，这个变量是在com/xxx/xxx包下的HelperProxy类下面的onClick方法的实例。这个onclick方法的描述是(Landroid/view/View;)V，这句话具体来说是什么意思呢，括号内的Landroid/view/View;代表这个方法的参数是一个android/view/View类型的变量。后面的v代表这个方法本身是一个void类型。那么你可能会说了，光知道这一个，我也不知道别的怎么写啊。我这里大致说一下一些基本类型的写法，关于其他没写到的用法，我会贴出ASM4的pdf说明文档，里面会有详尽说明。
![4.png](E:/222/4.png "")
 

以上是一些基本数据类型的用法，下面是一些使用案例：
![5.png](E:/222/5.png "")
 
这样，大家就能了解一些基本的ASM库下的字节码写法。

通过上面的描述，我们就实现了一个最简单的onclick方法的字节码插桩。

自定义gradle插件的限制
讲完了基本实现步骤，最后，我这里再来谈谈自定义gradle的一些限制。虽然他的功能很强大，但是它的使用场景，其实主要还是针对项目代码。因为gradle插件的transform接口，本身只会暴露在编译过程当中产生的字节码，其中包括了你的项目代码，你的aar包代码，你的jar包代码。也就是说，自定义gradle插件并不具备hook系统代码的能力。如果说你的目标是hook系统代码，比如给某些系统接口设置自己的代理实现类，那么自定义gradle插件并不是你的可行选择。

##关于作者
郑逸钧，来自某支付公司，资深android开发。个人主页：http://engineerblog.cn


