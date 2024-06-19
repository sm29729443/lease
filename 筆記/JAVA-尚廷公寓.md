# 項目目的

- 1. 學習一些 JAVA 的開發技巧。
- 2. 想知道 redis 實際上會應用在什麼場景。

對於此項目的初始化步驟這裡不做筆記，直接看影片教學文檔即可。

## BaseEntity

module:model

package:com.atguigu.lease.model.entity

項目中將 entity 的常用字段(id`、`create_time`、`update_time`、`is_deleted`)給抽取到了一個 class BaseEntity，並且讓其他 entity 去 extends BaseEntity。

優點:方便管理，假設今天要對 `is_deleted` 這個屬性加上一些 annotation，那只要在 BaseEntity 加上即可。

## 資料庫的狀態字段在 JAVA 中使用 enums

對於資料庫中的狀態字段，通常是儲存一個 int，而非直接儲存字串，譬如訂單狀態，在資料庫中並不會直接存者「待支付、待發貨、待收貨」等等，而是存者「1、2、3 ...」去表示訂單的狀態，但如果在 JAVA 中也用 INT 去寫程式碼的話，那可讀性很差 ，譬如:

```java
order.setStatus(1);
if (order.getStatus() == 1) {
order.setStatus(2);
}
```

上述這種程式碼的可讀性很差，此時可以使用 enums。

```java
public enum Status {

CANCEL(0, "已取消"),
WAIT_PAY(1, "待支付"),
WAIT_TRANSFER(2, "待发货"),
WAIT_RECEIPT(3, "待收货"),
RECEIVE(4, "已收货"),
COMPLETE(5, "已完结");

private final Integer value;
private final String desc;

public Integer value() {
  return value;
}
public String desc() {
  return desc;
}
}
```

```java
@Data
public class Order{
 // 省略其他字段 ...
 private Status status;
 ...
}
```

這樣程式碼就會變成如下，可讀性就會上升。

```java
order.setStatus(Status.WAIT_PAY);
```

## 將 entity 序列化

可以看到項目中的 `BaseEntity implements Serializable`，教學是說如果想將某個 obj 放到緩存(如 redis)，那就得 `implements Serializable`。

## MyBatis-Plus 提供的邏輯刪除功能

因為此項目都是邏輯刪除，而假設今天想查詢資料庫中的資料，譬如查詢全部的支付方式，那當然是只能查詢 `is_deleted = 0`的資料，因此實作方式就會變為:

```java
    // api url:/admin/payment/list
    @GetMapping("list")
    public Result<List<PaymentType>> listPaymentType() {
        LambdaQueryWrapper<PaymentType> lambdaQueryWrapper = new LambdaQueryWrapper();
        lambdaQueryWrapper.eq(PaymentType::getIsDeleted, 0);
        List<PaymentType> paymentTypeList = paymentTypeService.list(lambdaQueryWrapper);
        return Result.ok(paymentTypeList);
    }
```

但如果用上述這種方式，那就得每個查詢都要自己加上QueryWrapper，工作量不小，因此 MyBatis-Plus 提供了一種邏輯刪除功能，可以自動的為 sql 語句添加 `is_deleted = 0`的 where 過濾語句。

### MyBatis-Plus 邏輯刪除功能步驟

#### Step.1

此步驟有兩種實現方式

##### 方式1

在`application.yml`新增以下內容

```yml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: flag # 指定要用來判斷邏輯刪除的實體類字段名，小心不是資料庫的字段名，而是映射的 java entity 屬性名
```

##### 方式2

使用 annotation `@TableLogic`

```java
    @TableLogic
    @TableField("is_deleted")
    private Byte isDeleted;
```

#### step.2

在 `application.yml`配置邏輯刪除的判斷值

```yml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-value: 1 # 邏輯已刪除值(默認值為 1)
      logic-not-delete-value: 0 # 邏輯未刪除值(默認值為 2)
```

#### result

可以看到 sql 語句自訂加上了 where is_deleted = 0 的判斷

<img src="img/Snipaste_2024-06-19_17-37-29.jpg" alt="mb邏輯刪除sql語句" style="width:100%"/>
