# lambda

## 跟据某个属性分组OfficeId

```java
Map<String, List<IncomeSumPojo>> collect =
    list.stream().collect(Collectors.groupingBy(IncomeSumPojo::getOfficeId));
```

## 根据某个属性分组OfficeId，汇总某个属性Money

```java
Map<String, Double> collect = list.stream().collect(
    Collectors.groupingBy(IncomeSumPojo::getOfficeId,Collectors.summingDouble(IncomeSumPojo::getMoney)));
```

## 根据某个属性添加条件过滤数据

```java
list = list.stream().filter(u -> !u.getAmount().equals("0.00")).collect(Collectors.toList());
```

## 判断一组对象里面有没有属性值是某个值

```java
List<Menu> menuList = UserUtils.getMenuList();
boolean add = menuList.stream().anyMatch(m -> "plan:ctPlan:add".equals(m.getPermission()));
```

## 取出一组对象的某个属性组成一个新集合

```java
List<String> tableNames=list.stream().map(User::getMessage).collect(Collectors.toList());
```

## 去重

```java
List<User> res = distinctList.stream().collect(//list是需要去重的list，返回值是去重后的list
        Collectors.collectingAndThen(Collectors.toCollection(() -> new TreeSet<>(Comparator.comparing(o -> o.getId()))), ArrayList::new));
```