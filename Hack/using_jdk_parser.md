Title: 使用JDK的Parser来解析Java源代码
Date: 2015-04-12
Slug: using-jdk-parser

在JDK中，自带了一套相关的编译API，可以在Java中发起编译流程，解析Java源文件然后获取其语法树，在JDK的``tools.jar``（OSX下可以在/Library/Java/JavaVirtualMachines/jdk_version/Contents/Home/lib中找到）中包含着这整套API，但是这却不是Oracle和OpenJDK发布中的公开API，因此对于这套API，并没有官方的正式文档来进行说明。但是，也有不少项目利用了这套API来做了不少事情，例如大名鼎鼎的[lombok](https://github.com/rzwitserloot/lombok.git)使用了这套API在Annotation Processing阶段修改了源代码中的语法树，最终结果相当于直接在源文件中插入了新的代码！

由于这套API目前缺少相关文档，使用起来比较困难，例如，解析源代码中的所有变量，并打印出来：
```
public class JavaParser {
    
    private static final String path = "User.java";
    
    private JavacFileManager fileManager;
    private JavacTool javacTool;
    
    public JavaParser() {
        Context context = new Context();
        fileManager = new JavacFileManager(context, true, Charset.defaultCharset());
        javacTool = new JavacTool();
    }
    
    public void parseJavaFiles() {
        Iterable<? extends JavaFileObject> files = fileManager.getJavaFileObjects(path);
        JavaCompiler.CompilationTask compilationTask = javacTool.getTask(null, fileManager, null, null, null, files);
        JavacTask javacTask = (JavacTask) compilationTask;
        try {
            Iterable<? extends CompilationUnitTree> result = javacTask.parse();
            for (CompilationUnitTree tree : result) {
                tree.accept(new SourceVisitor(), null);
                
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    static class SourceVisitor extends TreeScanner<Void, Void> {

        private String currentPackageName = null;
        
        @Override
        public Void visitCompilationUnit(CompilationUnitTree node, Void aVoid) {        
            return super.visitCompilationUnit(node, aVoid);
        }

        @Override
        public Void visitVariable(VariableTree node, Void aVoid) {
            formatPtrln("variable name: %s, type: %s, kind: %s, package: %s", 
                    node.getName(), node.getType(), node.getKind(), currentPackageName);
            return null;
        }
    }
    
    public static void formatPtrln(String format, Object... args) {
        System.out.println(String.format(format, args));
    }
    
    public static void main(String[] args) {
        
        new JavaParser().parseJavaFiles();
    }
}
```

其中 ``User.java``的代码如下：
```
package com.ragnarok.javaparser;

import com.sun.istack.internal.Nullable;
import java.lang.Override;

public class User {
    
    @Nullable
    private String foo = "123123";
    private Foo a;
    
    public void UserMethod() {}
    
    static class Foo {
        private String fooString = "123123";
        
        public void FooMethod() {}
    }
}
```

执行上面的``JavaParser``结果如下：
```
variable: foo, annotaion: Nullable
variable name: foo, type: String, kind: VARIABLE, package: com.ragnarok.javaparser
variable name: a, type: Foo, kind: VARIABLE, package: com.ragnarok.javaparser
```

这里我们是首先通过``JavaCompiler.CompilationTask``解析了源文件之后，再使用自定义的``SourceVisitor``（继承自``TreeScanner``）来对源代码的结构进行访问，在``SourceVisitor``类中，通过重载``visitVariable``来对一个编译单元（单个源代码文件）进行解析，访问其中的所有的变量，这里可以看出，我们没有办法拿到这个变量类型的全限定名（包含包名），只能拿到的对应的简单名字，因此，类型的确定需要外部实现**自行确定**，例如可以通过记录类所在的包名，递归的搜索整个源代码目录来跟踪所有类的全限定名，查找import中是否包含对应的类型等。

``TreeScanner``中除了``visitVariable``方法外，还包含了大量其他的``visitXYZ``方法，例如，可以遍历所有的import，方法定义，Annotation等，更具体可以查看OpenJDK中关于这个的源代码

这里再来看下另外一个例子，重载``visitClass``方法，访问所有的内部类以及类本身：

```
@Override
public Void visitClass(ClassTree node, Void aVoid) {
    formatPtrln("class name: %s", node.getSimpleName());
    for (Tree member : node.getMembers()) {
        if (member instanceof VariableTree) {
            VariableTree variable = (VariableTree) member;
            List<? extends AnnotationTree> annotations = variable.getModifiers().getAnnotations();
            if (annotations.size() > 0) {
                formatPtrln("variable: %s, annotaion: %s", variable.getName(), annotations.get(0).getAnnotationType());
            } else {
                formatPtrln("variable: %s", variable.getName());
            }                
        }
    }
    return super.visitClass(node, aVoid);
 }
```
这里简单的打印了类名以及变量的名称，类型，annotation类型，执行上面的代码，结果如下：

```
class name: User
variable: foo, annotaion: Nullable
variable: a
class name: Foo
variable: fooString
```
可以看出我们把类名以及类中的变量都打印了出来。而在``visitClass``方法中，我们可以通过``getMembers``方法拿到类中所有的成员，包括变量，方法，annotation等，分别对应着不同的类型，例如变量就对应着``VariableTree``类型，方法就对应的``MethodTree``类型。

 ----
 
 总得来说，虽然实际上使用并不算特别复杂，但是由于缺少文档，对使用造成了很大的障碍，而且目前所介绍的只是这套API的一少部分，后续我将会继续研究这套API的相关函数。