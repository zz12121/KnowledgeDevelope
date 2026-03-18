# SOLID 七大设计原则

> 设计原则是设计模式背后的理论基础，是指导软件系统保持灵活、可扩展、低耦合的黄金法则。共七条，前五条合称 SOLID。本文通过生动的小故事来引入每个原则，让抽象的概念变得通俗易懂。

---

## 故事引入

### 1. 单一职责原则 —— 《一个小老板的发家史》

阿明想开一家早餐店卖煎饼，招收了100名年轻人。他做了如下安排：
- 20人去学和面
- 20人去学制馅
- 20人去学包馅
- 20人去学成型
- 20人去学烘焙

分工明确，各司其职，这就是**单一职责**！如果让一个人同时负责和面、制馅、包馅，那这个人就太累了，稍微有点变动就会影响全局。

### 2. 里氏替换原则 —— 《我买了宝马，为啥不让我停这？》

我们怎么识别一辆汽车是宝马品牌？只要是宝马汽车，车头一定会有宝马的logo。

宝马公司跟各地负责宝马专属停车位的保安说："只允许车辆有宝马logo的汽车进入BMW的免费停车场。"

这里有个问题：如果有人开着一辆"宝马"外观的车，但没有宝马logo，保安该让进吗？

**里氏替换原则**告诉我们：子类可以扩展父类的功能，但不能改变父类已有的功能。保安（客户端）只认"有宝马logo"这个特征（父类/接口的契约），不符合这个特征的车（子类）就不能替换进去。

### 3. 依赖倒置原则 —— 《我国出租车行业发展史》

出租车司机的培养需要两个关键因素：
- 合格的驾驶员
- 出租车

那么出租车应该具备哪些特点呢？
- 百公里油耗要小（省油）
- 车子得"皮实"，不能总坏
- 车子要大，能装下足够的客户
- 车子价格要便宜，不能高于15万

**依赖倒置**的核心是：高层模块不应该依赖低层模块，两者都应该依赖抽象。就像"打车软件"不应该依赖具体某款车，而是依赖"汽车"这个抽象概念，这样换成什么车都能用。

### 4. 接口隔离原则 —— 《做个Rapper咋这么难？》

什么才是真正的rapper？不同人可能有不同的定义：
- 说唱功底很厉害
- 唱功也一定要厉害
- 要会freestyle
- 需要自己写词写曲
- 穿衣服一定要又潮又帅
- 要有纹身
- ……

如果把这些"能力"都塞进一个"Rapper接口"，那么一个只想好好唱歌的歌手也得会freestyle、会写词、有纹身——这太痛苦了！

**接口隔离**告诉我们：接口要尽量小，不要建立臃肿庞大的接口。把"会唱歌"和"会freestyle"分成两个接口，让需要的人各取所需。

### 5. 迪米特法则 —— 《只是买台咖啡机，竟然要学习咖啡器的运行原理？》

你去买咖啡机，店员给你一本厚厚的说明书："想用这台咖啡机？你得先学会咖啡豆的烘焙原理、萃取温度控制、水质矿物质配比……"

你内心OS：我只是想打个咖啡喝，要不要这么复杂？

**迪米特法则**告诉我们：一个类应该对自己需要耦合或调用的类知道得最少。咖啡机把复杂操作封装成简单的"按键即可"，你不需要知道内部原理。

### 6. 开闭原则 —— 《我发誓！再也不买一体机了》

你想玩《逃离塔科夫》这款游戏，发现苹果一体机根本跑不动！你后悔极了：买的时候以为性能足够，结果游戏要求更高，升级都没办法升级——因为一体机无法扩展硬件！

**开闭原则**告诉我们：类、方法、模块应该对扩展开放，对修改关闭。买电脑就应该买组装机——想升级显卡就换显卡，想加内存就加内存，而不是整机更换。

---

## 一、单一职责原则（SRP）

> **一个类应该只有一个引起它变化的原因。**

### 核心理解

职责就是变化的原因。如果一个类承担了多个职责，当其中任何一个职责的需求发生变化，都可能影响该类中不相关的其他功能，导致系统脆弱。

判断方法：问自己"有多少个不同的业务变化会导致这个类被修改"，如果超过一个，职责就过多了。

### Demo

```java
// ❌ 违反SRP：一个类干了持久化、验证、邮件三件事
public class UserManager {
    public void saveUser(User user) {
        // 操作数据库，持久化逻辑
    }
    public boolean validateUser(User user) {
        // 格式验证逻辑
        return user.getEmail().contains("@");
    }
    public void sendWelcomeEmail(User user) {
        // 发送邮件逻辑
    }
    // 改数据库结构、改验证规则、改邮件内容，都要改这个类 ——— 三个变化原因！
}

// ✅ 遵循SRP：每个类只有一个职责
public class UserRepository {
    public void save(User user) {
        // 只负责数据库持久化
    }
    public User findById(Long id) { /* ... */ }
}

public class UserValidator {
    public ValidationResult validate(User user) {
        // 只负责业务规则验证
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            return ValidationResult.fail("邮箱格式不正确");
        }
        return ValidationResult.success();
    }
}

public class UserEmailService {
    public void sendWelcomeEmail(User user) {
        // 只负责邮件发送
    }
    public void sendPasswordResetEmail(User user, String resetLink) { /* ... */ }
}
```

### 实际业务场景

```java
// 场景：订单服务拆分
// ❌ 一个 OrderService 负责所有事
public class OrderService {
    public void createOrder(Order order) { /* 创建 */ }
    public void calculateDiscount(Order order) { /* 计算折扣 */ }
    public void sendConfirmationEmail(Order order) { /* 发邮件 */ }
    public void generateInvoice(Order order) { /* 生成发票 */ }
    public void updateInventory(Order order) { /* 更新库存 */ }
}

// ✅ 按职责拆分，通过事件或注入协调
@Service
public class OrderCreationService {
    private final OrderRepository orderRepository;
    private final DiscountCalculator discountCalculator;
    private final ApplicationEventPublisher eventPublisher;

    public Order createOrder(CreateOrderRequest request) {
        BigDecimal discount = discountCalculator.calculate(request);
        Order order = Order.create(request, discount);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}

@Service
public class DiscountCalculator { /* 只负责折扣计算 */ }

@EventListener
@Service
public class OrderNotificationService { /* 只负责发邮件 */ }

@EventListener
@Service
public class InventoryUpdateService { /* 只负责更新库存 */ }
```

---

## 二、开闭原则（OCP）

> **软件实体应该对扩展开放，对修改关闭。**

### 核心理解

当需求变化时，应该通过**新增代码**来扩展功能，而不是修改已有代码。修改已有代码会引入 bug 风险，而新增代码（新文件、新实现类）风险更低。

实现开闭原则的关键：**抽象出变化点，依赖抽象而非具体**。

### Demo

```java
// ❌ 违反OCP：每加一种图形就要改这个方法
public class AreaCalculator {
    public double calculate(Object shape) {
        if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.getRadius() * c.getRadius();
        } else if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.getWidth() * r.getHeight();
        }
        // 新增三角形？必须修改此处，违反OCP
        throw new IllegalArgumentException("Unknown shape");
    }
}

// ✅ 遵循OCP：抽象固定，扩展用新类
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }

    @Override
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double width, double height) {
        this.width = width; this.height = height;
    }

    @Override
    public double area() { return width * height; }
}

// 新增三角形：只加一个类，不改任何已有代码
public class Triangle implements Shape {
    private final double base, height;
    public Triangle(double base, double height) {
        this.base = base; this.height = height;
    }

    @Override
    public double area() { return 0.5 * base * height; }
}

// 计算器永远不需要改
public class AreaCalculator {
    public double totalArea(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

### 实际业务场景

```java
// 场景：报表导出格式扩展（策略模式实现OCP）
public interface ReportExporter {
    byte[] export(ReportData data);
    String getSupportedFormat();
}

@Component
public class ExcelExporter implements ReportExporter {
    public byte[] export(ReportData data) { /* 导出Excel */ return new byte[0]; }
    public String getSupportedFormat() { return "excel"; }
}

@Component
public class PdfExporter implements ReportExporter {
    public byte[] export(ReportData data) { /* 导出PDF */ return new byte[0]; }
    public String getSupportedFormat() { return "pdf"; }
}

// 新增CSV导出：只加一个类
@Component
public class CsvExporter implements ReportExporter {
    public byte[] export(ReportData data) { /* 导出CSV */ return new byte[0]; }
    public String getSupportedFormat() { return "csv"; }
}

// 调度器：无需修改
@Service
public class ReportService {
    private final Map<String, ReportExporter> exporters;

    @Autowired
    public ReportService(List<ReportExporter> exporterList) {
        exporters = exporterList.stream()
            .collect(Collectors.toMap(ReportExporter::getSupportedFormat, e -> e));
    }

    public byte[] export(ReportData data, String format) {
        return Optional.ofNullable(exporters.get(format))
            .orElseThrow(() -> new UnsupportedOperationException("不支持的格式: " + format))
            .export(data);
    }
}
```

---

## 三、里氏替换原则（LSP）

> **所有引用父类的地方，都必须能透明地替换为子类，程序行为不变。**

### 核心理解

里氏替换是对继承的约束。子类可以扩展父类，但不能改变父类已经定义的行为契约。违反 LSP 意味着继承关系在语义上是错的。

具体规则：
- 子类方法的前置条件（输入要求）不能比父类更严格
- 子类方法的后置条件（输出承诺）不能比父类更宽松
- 子类不能抛出父类没有声明的异常

### Demo

```java
// ❌ 违反LSP的经典案例：正方形继承矩形
class Rectangle {
    protected int width, height;

    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int getArea() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        super.setWidth(w);
        super.setHeight(w); // 设置宽同时改高，破坏了Rectangle的行为契约
    }

    @Override
    public void setHeight(int h) {
        super.setHeight(h);
        super.setWidth(h);  // 设置高同时改宽，破坏了Rectangle的行为契约
    }
}

// 这个测试方法对Rectangle通过，对Square失败 ——— 违反了LSP
void testArea(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    // 传Rectangle：area = 20 ✅
    // 传Square：width=4,height=4，area = 16 ❌
    assert r.getArea() == 20 : "期望20，实际" + r.getArea();
}

// ✅ 正确做法：两者有公共行为，用共同接口而非继承
public interface Shape {
    double area();
}

public class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double width, double height) {
        this.width = width; this.height = height;
    }
    public double area() { return width * height; }
}

public class Square implements Shape {
    private final double side;
    public Square(double side) { this.side = side; }
    public double area() { return side * side; }
}
```

### 实际业务场景

```java
// 场景：支付处理器继承体系
public abstract class PaymentProcessor {
    // 契约：处理支付，成功返回交易号，失败抛出PaymentException
    public abstract String processPayment(BigDecimal amount) throws PaymentException;

    // 具体方法：日志记录
    protected void logPayment(BigDecimal amount, String result) {
        System.out.println("支付处理：金额=" + amount + "，结果=" + result);
    }
}

// ✅ AlipayProcessor：遵守契约，正常返回交易号或抛PaymentException
public class AlipayProcessor extends PaymentProcessor {
    @Override
    public String processPayment(BigDecimal amount) throws PaymentException {
        // 调用支付宝API
        String tradeNo = callAlipayAPI(amount);
        logPayment(amount, tradeNo);
        return tradeNo;
    }
    private String callAlipayAPI(BigDecimal amount) throws PaymentException {
        return "alipay-" + System.currentTimeMillis();
    }
}

// ❌ 违反LSP的子类：抛出了父类没有声明的异常
public class BadPaymentProcessor extends PaymentProcessor {
    @Override
    public String processPayment(BigDecimal amount) {
        // 抛出了父类没有声明的RuntimeException，调用方可能没有处理这个异常
        throw new NullPointerException("非法操作"); // 违反LSP！
    }
}
```

---

## 四、接口隔离原则（ISP）

> **客户端不应该被迫依赖它不使用的方法；接口应该小而专。**

### 核心理解

把大而全的接口拆分成小而专的接口，使得每个接口只包含特定角色需要的方法。这样客户端只需要依赖自己用到的接口，减少不必要的耦合。

### Demo

```java
// ❌ 违反ISP：一个臃肿的接口
public interface Animal {
    void eat();
    void sleep();
    void fly();   // 不是所有动物都能飞
    void swim();  // 不是所有动物都能游泳
    void run();   // 不是所有动物都能跑
}

// 企鹅被迫实现飞行方法
public class Penguin implements Animal {
    public void eat() { System.out.println("企鹅吃鱼"); }
    public void sleep() { System.out.println("企鹅睡觉"); }
    public void fly() { throw new UnsupportedOperationException("企鹅不会飞！"); }
    public void swim() { System.out.println("企鹅游泳"); }
    public void run() { System.out.println("企鹅走路"); }
}

// ✅ 遵循ISP：按能力拆分接口
public interface Eatable { void eat(); }
public interface Sleepable { void sleep(); }
public interface Flyable { void fly(); }
public interface Swimmable { void swim(); }
public interface Runnable { void run(); }

// 每种动物只实现它具备的能力
public class Eagle implements Eatable, Sleepable, Flyable, Runnable {
    public void eat() { System.out.println("鹰吃猎物"); }
    public void sleep() { System.out.println("鹰休息"); }
    public void fly() { System.out.println("鹰翱翔天空"); }
    public void run() { System.out.println("鹰奔跑"); }
}

public class Penguin implements Eatable, Sleepable, Swimmable, Runnable {
    public void eat() { System.out.println("企鹅吃鱼"); }
    public void sleep() { System.out.println("企鹅睡觉"); }
    public void swim() { System.out.println("企鹅游泳"); }
    public void run() { System.out.println("企鹅小跑"); }
    // 没有 fly 方法，完全合理
}
```

### 实际业务场景

```java
// 场景：用户权限接口拆分
// ❌ 臃肿的用户接口
public interface UserOperations {
    User findById(Long id);
    void updateProfile(User user);
    void changePassword(String oldPwd, String newPwd);
    List<User> findAllUsers();       // 管理员功能
    void deleteUser(Long id);        // 管理员功能
    void lockAccount(Long userId);   // 管理员功能
}

// ✅ 按角色隔离接口
public interface UserQueryService {
    User findById(Long id);
    User findByUsername(String username);
}

public interface UserProfileService {
    void updateProfile(Long userId, UserProfileRequest request);
    void changePassword(Long userId, ChangePasswordRequest request);
}

public interface UserAdminService {
    List<User> findAllUsers(PageRequest pageRequest);
    void deleteUser(Long userId);
    void lockAccount(Long userId);
    void unlockAccount(Long userId);
}

// 普通用户只注入自己需要的接口
@Service
public class PersonalCenterController {
    @Autowired private UserQueryService queryService;
    @Autowired private UserProfileService profileService;
    // 没有 AdminService，普通用户不能调管理功能
}

// 管理员控制器注入所有接口
@Service
public class AdminController {
    @Autowired private UserQueryService queryService;
    @Autowired private UserAdminService adminService;
}
```

---

## 五、依赖倒转原则（DIP）

> **高层模块不应该依赖低层模块，两者都应该依赖抽象；抽象不应该依赖细节，细节应该依赖抽象。**

### 核心理解

传统设计里高层调用低层，依赖方向自上而下。DIP 把这个关系"倒转"——高层和低层都依赖中间的抽象接口，接口由高层定义，低层实现依赖于高层定义的接口规范。

实现 DIP 的主要方式是依赖注入（Dependency Injection），Spring 的 IoC 容器就是依赖注入的最佳实践。

### Demo

```java
// ❌ 违反DIP：高层直接 new 低层
public class OrderService {
    // 直接依赖具体实现，更换数据库实现时需要改这里
    private MySQLOrderRepository repository = new MySQLOrderRepository();

    public void createOrder(Order order) {
        repository.save(order);
    }
}

// ✅ 遵循DIP：高层依赖接口，低层实现接口
public interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
    List<Order> findByUserId(Long userId);
}

// 低层1：MySQL实现
@Repository
public class MySQLOrderRepository implements OrderRepository {
    @Autowired private JdbcTemplate jdbcTemplate;

    @Override
    public void save(Order order) {
        jdbcTemplate.update("INSERT INTO orders ...", order.getId(), ...);
    }

    @Override
    public Order findById(Long id) { /* MySQL查询 */ return null; }

    @Override
    public List<Order> findByUserId(Long userId) { /* MySQL查询 */ return List.of(); }
}

// 低层2：MongoDB实现（切换数据库只需换实现，高层不动）
@Repository
public class MongoOrderRepository implements OrderRepository {
    @Autowired private MongoTemplate mongoTemplate;

    @Override
    public void save(Order order) { mongoTemplate.save(order); }

    @Override
    public Order findById(Long id) { return mongoTemplate.findById(id, Order.class); }

    @Override
    public List<Order> findByUserId(Long userId) {
        return mongoTemplate.find(Query.query(Criteria.where("userId").is(userId)), Order.class);
    }
}

// 高层：只依赖接口，不关心具体实现
@Service
public class OrderService {
    private final OrderRepository orderRepository; // 依赖抽象

    @Autowired // Spring 注入具体实现
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        orderRepository.save(order);
        return order;
    }
}
```

---

## 六、迪米特法则（LoD / 最少知识原则）

> **一个对象应该对其他对象保持最少的了解；只和直接朋友通信，不和陌生人说话。**

### 核心理解

"朋友"指的是：当前对象本身、方法参数、成员变量引用的对象、方法内创建的对象。凡是通过"中间人"传递才能访问到的对象，都是"陌生人"，不该直接调用。

违反迪米特法则的典型症状：`a.getB().getC().doSomething()` 这种"火车式调用"。

### Demo

```java
// ❌ 违反迪米特法则：Customer 直接操作 Address 的 City
public class CustomerService {
    public void processOrder(Long customerId) {
        Customer customer = customerRepo.findById(customerId);
        // 深入到陌生人 city，CustomerService 不该知道 Address 的内部结构
        String cityName = customer.getAddress().getCity().getName();
        if ("北京".equals(cityName)) {
            // 北京客户特殊处理
        }
    }
}

// ✅ 遵循迪米特法则：每层封装自己的逻辑
public class Customer {
    private Address address;

    // Customer 提供高层方法，隐藏内部结构
    public String getCityName() {
        return address.getCityName(); // Customer 和 Address 是朋友
    }

    public boolean isFromCity(String cityName) {
        return cityName.equals(getCityName());
    }
}

public class Address {
    private City city;

    public String getCityName() {
        return city.getName(); // Address 和 City 是朋友
    }
}

// CustomerService 只和直接朋友 Customer 交互
public class CustomerService {
    public void processOrder(Long customerId) {
        Customer customer = customerRepo.findById(customerId);
        if (customer.isFromCity("北京")) { // 不再深入内部
            // 北京客户特殊处理
        }
    }
}
```

### 实际业务场景

```java
// 场景：订单金额计算封装
// ❌ 业务层深入到底层获取每个 Item 的价格信息
public class OrderCalculator {
    public BigDecimal calculateTotal(Order order) {
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItem item : order.getItems()) {
            // 深入到 item.getProduct().getPrice() ——— 陌生人
            BigDecimal price = item.getProduct().getPrice();
            int quantity = item.getQuantity();
            total = total.add(price.multiply(new BigDecimal(quantity)));
        }
        return total;
    }
}

// ✅ 让 OrderItem 封装自己的小计计算
public class OrderItem {
    private Product product;
    private int quantity;

    // OrderItem 自己负责计算小计，不暴露内部 Product
    public BigDecimal getSubtotal() {
        return product.getPrice().multiply(new BigDecimal(quantity));
    }
}

public class OrderCalculator {
    public BigDecimal calculateTotal(Order order) {
        // 只和 OrderItem 交互，不接触陌生人 Product
        return order.getItems().stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

---

## 七、合成复用原则（CRP）

> **优先使用组合/聚合，而不是继承来达到代码复用的目的。**

### 核心理解

继承是"白箱复用"——子类能看见父类所有实现细节，高度耦合，且继承关系在编译期固定。

组合是"黑箱复用"——只通过接口交互，被组合对象内部对外不可见，耦合度低，且可以在运行时动态替换。

继承应该用于真正的"is-a"关系（子类是父类的一种特化），而不是为了复用几行代码就随便继承。

### Demo

```java
// ❌ 错误使用继承复用：Stack 继承 Vector 是个历史遗留错误
// Stack 只应该有 push/pop/peek，但继承了 Vector 的 add/get/remove 等所有方法
// 导致 Stack 可以被当成 Vector 随意插入/删除，破坏了栈的语义
public class MyStack<T> extends Vector<T> {
    public T push(T item) { addElement(item); return item; }
    public T pop() { T obj = peek(); removeElementAt(size() - 1); return obj; }
    public T peek() { return elementAt(size() - 1); }
}

// ✅ 正确做法：用组合实现栈
public class BetterStack<T> {
    private final List<T> elements = new ArrayList<>(); // 组合，不继承

    public void push(T item) { elements.add(item); }

    public T pop() {
        if (isEmpty()) throw new EmptyStackException();
        return elements.remove(elements.size() - 1);
    }

    public T peek() {
        if (isEmpty()) throw new EmptyStackException();
        return elements.get(elements.size() - 1);
    }

    public boolean isEmpty() { return elements.isEmpty(); }
    public int size() { return elements.size(); }
    // 注意：没有 add(index, item) 这种破坏栈语义的方法
}
```

### 实际业务场景

```java
// 场景：日志功能复用，用组合而非继承
// ❌ 通过继承复用日志功能（把所有 Service 都继承 BaseLoggingService）
public class BaseLoggingService {
    protected void log(String message) { System.out.println("[LOG] " + message); }
}

public class OrderService extends BaseLoggingService {
    public void createOrder(Order order) {
        log("创建订单: " + order.getId()); // 复用了日志，但 OrderService 现在是 BaseLoggingService 的子类
    }
}
// 如果 OrderService 还需要继承另一个类，Java 单继承就冲突了

// ✅ 通过组合复用日志功能（任意类都可以使用，不影响继承关系）
public interface Logger {
    void log(String level, String message);
}

@Component
public class ConsoleLogger implements Logger {
    public void log(String level, String message) {
        System.out.printf("[%s] %s%n", level, message);
    }
}

@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final Logger logger; // 组合：注入 Logger，不继承

    @Autowired
    public OrderService(OrderRepository orderRepository, Logger logger) {
        this.orderRepository = orderRepository;
        this.logger = logger;
    }

    public Order createOrder(CreateOrderRequest request) {
        logger.log("INFO", "开始创建订单");
        Order order = Order.from(request);
        orderRepository.save(order);
        logger.log("INFO", "订单创建成功: " + order.getId());
        return order;
    }
}
```

---

## 八、七大原则关系总结

```
           开闭原则（目标）
               ↑
    ┌──────────┼──────────┐
    │          │          │
依赖倒转     里氏替换    接口隔离
（手段）     （约束继承） （细化抽象）
    │
合成复用（实现方式：优先组合）
    │
单一职责（划清边界，让每个模块只有一个变化原因）
    │
迪米特法则（控制知识传播，降低耦合）
```

七条原则的协作：**开闭**是终极目标（软件能扩展，不修改）；**依赖倒转**是实现手段（依赖抽象）；**里氏替换**保证抽象可安全替换；**接口隔离**让抽象保持精准；**单一职责**保证每个模块边界清晰；**迪米特**减少模块间不必要的知识传递；**合成复用**在具体实现上优先用组合代替继承。

## 九、与 [[../../31_KafkaKnowledge/索引]] 的横向知识连接

- 七大原则是 GoF 23 种设计模式的理论基础，详见 [[../02_创建型模式/01、创建型模式详解]]
- 在 Spring 框架源码中，这七条原则无处不在，详见 [[../05_综合实战/01、框架源码与实战]]
