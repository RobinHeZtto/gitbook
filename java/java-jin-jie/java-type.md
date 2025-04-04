# Java类型系统

> 泛型引入之前，Java中的类型系统很简单，完全取决于Class，泛型引入后，变为以**java.lang.reflect.Type**为标识的更加复杂的类型系统。 &#x20;
>
> 我们知道Java中的泛型是用擦除法实现的，这样实现的问题就是 `ArrayList<Integer>`和`ArrayList<String>`对应的class都是`ArrayList.class，`仅仅以Class做类型标识不够了，为了解决这种问题，Type应运而生。\
>

**Type**是Java语言中所有类型的公共父接口，Type有四个子接口：ParameterizedType、 TypeVariable、GenericArrayType、WildcardType，及一个实现类：Class

![](<../../.gitbook/assets/image (394).png>)



![](<../../.gitbook/assets/image (489).png>)

* **Class：**&#x4E0E;之前的含义相同，代表原始类型
* **ParameterizedType参数化类型：**&#x6BD4;如List\<Integer>，List\<Double>，List\<T>在这里参数化指的就是 Integer和Double，T。
* **TypeVariable** **类型变量：**&#x6BD4;如参数化类型中的 **T，**&#x8868;示泛指任何类
* **GenericArrayType** **泛型数组类型**：比如List\<T>\[]，T\[]这种。表示**一种** **元素类型是参数化类\*\*\*\*型或者类型变量的**数组类型。
* **WildCardType通配符类型：** 表示通配符表达式，**它是Type的一个子接口，但是不作为Java中的类型，**&#x8868;示 `? extend Number`, `? super Number`,`?` 这种。

```
public interface Type {
    //返回这个类型的描述，包括此类型的参数描述
    default String getTypeName() {
        return toString();
    }
}
```



### **ParameterizedType**

&#x20;       ParameterizedType即参数化类型，即我们通常所说的泛型类型。那么参数化类型怎么理解呢？顾名思义，就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）。

```
public interface ParameterizedType extends Type {
   
    Type[] getActualTypeArguments();
  
    Type getRawType();

    Type getOwnerType();
}
```

         **getRawType(): Type**

&#x20;       该方法的作用是返回当前的ParameterizedType的类型。如一个List，返回的是List的Type，即返回当前参数化类型本身的Type。

&#x20;       **getOwnerType(): Type**

返回ParameterizedType类型所在的类的Type。如Map.Entry\<String, Object>这个参数化类型返回的事Map(因为Map.Entry这个类型所在的类是Map)的类型。

&#x20;       **getActualTypeArguments(): Type\[]**

&#x20;       该方法返回参数化类型<>中的实际参数类型， 如 Map\<String, Person> map 这个 ParameterizedType 返回的是 String 类, Person 类的全限定类名的 Type Array。**注意: 该方法只返回最外层的<>中的类型，无论该<>内有多少个<>。**

### **TypeVariable：**

&#x20;       TypeVariable即类型变量，范型信息在编译时会被转换为一个特定的类型, 而TypeVariable就是用来反映在JVM编译该泛型前的信息。(通俗的来说，TypeVariable就是我们常用的T，K这种泛型变量)。

*   **getBounds(): Type\[]:**

    返回当前类型的上边界，如果没有指定上边界，则默认为Object。
*   **getName(): String:**

    返回当前类型的类名
*   **getGenericDeclaration(): D**

    返回当前类型所在的类的Type。

### **GenericArrayType**

&#x20;       GenericArrayType即泛型数组类型，组成数组的元素中有泛型则实现了该接口; 它的组成元素是 ParameterizedType 或 TypeVariable 类型。(通俗来说，就是由参数类型组成的数组。如果仅仅是参数化类型，则不能称为泛型数组，而是参数化类型)。**注意：无论从左向右有几个\[]并列，这个方法仅仅脱去最右边的\[]之后剩下的内容就作为这个方法的返回值。**

* **getGenericComponentType(): Type:**\
  返回组成泛型数组的实际参数化类型，如List\[] 则返回 List。



### **WildcardType:**&#x20;

WildcardType即通配符类型，表示通配符类型，比如 \<?>,  \<? Extends Number>等

* **getLowerBounds(): Type\[]:** 得到下边界的数组
* **getUpperBounds(): Type\[]:** 得到上边界的type数组

**注：如果没有指定上边界，则默认为Object，如果没有指定下边界，则默认为String**
