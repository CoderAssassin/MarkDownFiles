[TOC]
工厂模式是干什么的？想象平时我们组装一台台式机，需要有主板，显卡，内存卡等各种零配件，当这些零配件齐了以后，接下来组装出来的才是一台完整的主机。类似的，java中创建对象的时候，很多情况下创建一个对象需要先创建好别的对象，然后该对象才能创建。然而我们不想自己动手来对其他那些对象进行创建，把这些工作交给“工厂”来完成，我们只要每次创建最终的对象就行，这就是工厂模式。
一共有三种工厂模式：**简单工厂模式，工厂方法模式和抽象工厂模式**。

## 简单工厂模式
``` java
public class SimpleComponentsPattern {

    private abstract class Components{
//        说明零件类型的方法
        public abstract void description();
    }

    private class Memory extends Components{
        @Override
        public void description() {
            System.out.println("这是内存!");
        }
    }

    private class CPU extends Components{
        @Override
        public void description() {
            System.out.println("这是CPU!");
        }
    }

    private class Graphics extends Components{
        @Override
        public void description() {
            System.out.println("这是显卡!");
        }
    }

    public static Components createComponent(String componentName){
        switch (componentName){
            case "memory":
                return new SimpleComponentsPattern().new Memory();
            case "cpu":
                return new SimpleComponentsPattern().new CPU();
            default:
                return new SimpleComponentsPattern().new Graphics();
        }
    }

    public static void main(String[] args){
        String componentName="cpu";
        Components component=SimpleComponentsPattern.createComponent(componentName);
        component.description();
    }
}
```
对于要创建的产品创建一个接口，不同的产品实现该接口然后在实现的接口方法中提供自己的方法实现，不同的具体的产品实现类的实例同一在一个工厂方法中进行创建，这样，只要通过该工厂来获取对应的产品实例就行。
简单工厂模式这里一共有三种对象，即**产品的接口或抽象类，具体产品类，以及工厂类**。
**缺陷**：扩展性差，当想增加一个组件的时候，需要重新创建一个组件类实现组件接口，然后再在工厂里添加对新的组件类的创建。这违背了开闭原则，即对扩展开放，对修改关闭，因为工厂方法每次都要修改。

## 工厂方法模式
前面说了，简单工厂方法模式的缺陷在于新增加一个组件的时候，每次都要修改工厂方法类的静态方法进行扩展。工厂方法模式主要是对工厂方法进行修改，使得工厂的创建方法不再是静态的方法，那么就可以通过继承该工厂类来创建新的工厂子类，子类里创建不同特色的产品。
工厂方法模式中有如下四中角色：**抽象工厂类或接口，具体工厂类，抽象产品类或接口，具体产品类**。
``` java
//组件接口
public interface IComponent {
    public void showMessage() throws Exception;
    public Map<String,Object> getMessage();
    public void setMessage(Map<String,Object> message);
}
//组件的具体实现类
public class Memory extends AbstractComponent {

    @Override
    public void showMessage() throws Exception {
        if (null==getMessage()||null==getMessage().get("componentName")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentName")+"!");
    }
}
public class CPU extends AbstractComponent {

    @Override
    public void showMessage() throws Exception {
        if (null==getMessage()||null==getMessage().get("componentName")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentName")+"!");
    }
}
public class Graphics extends AbstractComponent {

    @Override
    public void showMessage() throws Exception {
        if (null==getMessage()||null==getMessage().get("componentName")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentName")+"!");
    }
}
//工厂接口
public interface IComponentFatory {
    public IComponent createComponent(String componentName);
}
//抽象工厂类
public abstract class AbstractComponent implements IComponent{

    Map<String,Object> message;
    @Override
    public Map<String,Object> getMessage() {
        return this.message;
    }

    @Override
    public void setMessage(Map<String, Object> message) {
        this.message=message;
    }
}
//具体工厂实现类
 **/
public class ComponentFactory implements IComponentFatory{
    @Override
    public IComponent createComponent(String componentName) {

        IComponent component;
        Map<String,Object> message=new HashMap<>();

        switch (componentName){
            case "memory":
                component=new Memory();
                message.put("componentName","内存");
                break;
            case "cpu":
                component=new Memory();
                message.put("componentName","cpu");
                break;
            default:
                component=new Memory();
                message.put("componentName","显卡");
                break;
        }
        component.setMessage(message);
        return component;
    }
}
//客户端类
public class FactoryPatternTest {

    public static void main(String[] args) throws Exception {
        ComponentFactory componentFactory=new ComponentFactory();
        IComponent component;

        component=componentFactory.createComponent("memory");
        component.showMessage();
        component=componentFactory.createComponent("cpu");
        component.showMessage();
        component=componentFactory.createComponent("graphics");
        component.showMessage();
    }
}
```
工厂方法模式里抽象了工厂接口，这样可以实现不同的工厂类。也就是说，当有新的产品来的时候，我们可以通过产品实现一个新的产品实现类，然后再实现一个新的工厂类，作为对新的产品的处理，而不需要再原来的工厂实现类上做代码修改，实现了工厂的扩展性，符合开闭原则。

## 抽象工厂模式
工厂模式解决了工厂的扩展性问题，但是上边的产品都是一类，即组件，加入组件分为两类，一类分为系统组件，一类分为存储组件，那么就需要两个产品接口，分别对于系统组件和存储组件，他们都是属于组件。如果这个时候工厂类创建的组件来自同一类组件，那么这时候还是工厂方法模式，如果来自不同类，那么就是抽象工厂模式。
``` java
//组件接口
public interface IComponent {
    public void showMessage() throws Exception;
    public Map<String,Object> getMessage();
    public void setMessage(Map<String,Object> message);
}
//系统组件接口
public interface ISystemComponent extends IComponent{
    public void showSystemComponent() throws Exception;
}
//系统组件抽象类(这个主要是减少重复代码，可以没有)
public abstract class AbstractSystemComponent implements ISystemComponent{

    Map<String,Object> message;
    @Override
    public Map<String, Object> getMessage() {
        return this.message;
    }

    @Override
    public void setMessage(Map<String, Object> message) {
        this.message=message;
    }
}
//系统组件实现类，内存
public class Memory_System extends AbstractSystemComponent {

    Map<String,Object> message=new HashMap<>();

    @Override
    public void showSystemComponent() throws Exception {
        showMessage();
        if (null==getMessage()||null==getMessage().get("componentClass")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentClass")+"!");
    }

    @Override
    public void showMessage() throws Exception {
        if (null==getMessage()||null==getMessage().get("componentName")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentName")+"!");
    }
}
//存储组件接口
public interface ISaveComponent extends IComponent{
    public void showSaveComponent() throws Exception;
}
//抽象存储组件类
public abstract class AbstractSaveComponent implements ISaveComponent{

    Map<String,Object> message;
    @Override
    public Map<String, Object> getMessage() {
        return this.message;
    }

    @Override
    public void setMessage(Map<String, Object> message) {
        this.message=message;
    }
}
//存储组件实现类，usb
public class USB_Save extends AbstractSaveComponent{

    @Override
    public void showSaveComponent() throws Exception {
        showMessage();
        if (null==getMessage()||null==getMessage().get("componentClass")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentClass")+"!");
    }

    @Override
    public void showMessage() throws Exception {
        if (null==getMessage()||null==getMessage().get("componentName")||"".equals(getMessage().get("componentName")))
            throw new Exception("参数无效!");
        System.out.println("这是"+getMessage().get("componentName")+"!");
    }
}
//工厂接口
public interface IAbstractFactory {
    public ISystemComponent createSystemComponent();
    public ISaveComponent createSaveComponent();
}
//工厂实现类，配置为系统组件为内存，存储组件为usb
public class MyAbstractComponent implements IAbstractFactory {

    @Override
    public ISystemComponent createSystemComponent() {
        ISystemComponent systemComponent;
        Map<String,Object> message=new HashMap<>();
        message.put("componentClass","系统组件");
        systemComponent=new Memory_System();
        message.put("componentName","内存");
        systemComponent.setMessage(message);
        return systemComponent;
    }

    @Override
    public ISaveComponent createSaveComponent() {
        ISaveComponent saveComponent;
        Map<String,Object> message=new HashMap<>();
        message.put("componentClass","存储组件");
        saveComponent=new USB_Save();
        message.put("componentName","USB");
        saveComponent.setMessage(message);
        return saveComponent;
    }
}
//客户端实现类
public class AbstractFactoryPatternTest {

    public static void main(String[] args) throws Exception {
        IAbstractFactory factory=new MyAbstractComponent();
        factory.createSystemComponent().showSystemComponent();
        factory.createSaveComponent().showSaveComponent();
    }
}
```
抽象工厂模式主要是将产品分成不同的类，那么就需要提供不同类的接口和具体的类的实现，然后提供不同的工厂接口的实现类，每个实现类里提供不同类的产品的组合实现，这样通过不同的工厂实现类就能实现不同的产品组合，也就是说，工厂实现类里边就对产品进行了关联。
缺点：某类产品多一个实现类，那么就要改动很多的工厂实现类，因为组合的缘故。

## Reference
* [JAVA设计模式之工厂模式(简单工厂模式+工厂方法模式)](https://www.cnblogs.com/yumo1627129/p/7197524.html)
* [java 三种工厂模式](https://www.cnblogs.com/zailushang1996/p/8601808.html)