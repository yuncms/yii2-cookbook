# 数据表继承

在大多数关系数据库中都没有继承支持，因此如果需要，应该手动实现。
One of approaches to the problem is called [single table inheritance](http://martinfowler.com/eaaCatalog/singleTableInheritance.html),
described well by Martin Fowler.

根据实体表中的模式，我们添加了一个名为 `type` 的附加列，它决定了哪一个类将从一行数据中实例化。

我们用下面的类结构实现简单的车型继承：

```
Car
|- SportCar
|- HeavyCar
```

## 准备

我们将使用基本的应用程序。 创建和设置数据库后，执行以下SQL创建表并插入一些数据：

```sql
CREATE TABLE `car` (
    `id` int NOT NULL AUTO_INCREMENT,
    `name` varchar(255) NOT NULL,
    `type` varchar(255) DEFAULT NULL,
    PRIMARY KEY (`id`)
);

INSERT INTO car (id, NAME, TYPE) VALUES (1, 'Kamaz', 'heavy'), (2, 'Ferrari', 'sport'), (3, 'BMW', 'city');
```

现在用Gii生成 `Car` 模型。


## How to do it...

我们需要一个非常简单的自定义查询类，以便始终将车型应用于查询条件。 创建 `models/CarQuery.php`:

```php
namespace app\models;

use yii\db\ActiveQuery;

class CarQuery extends ActiveQuery
{
    public $type;
    public $tableName;

    public function prepare($builder)
    {
        if ($this->type !== null) {
            $this->andWhere(["$this->tableName.type" => $this->type]);
        }
        return parent::prepare($builder);
    }
}
```

现在我们为不同类型的汽车类创建模型。 首先 `models/SportCar.php`:

```php
namespace app\models;

class SportCar extends Car
{
    const TYPE = 'sport';

    public function init()
    {
        $this->type = self::TYPE;
        parent::init();
    }

    public static function find()
    {
        return new CarQuery(get_called_class(), ['type' => self::TYPE, 'tableName' => self::tableName()]);
    }

    public function beforeSave($insert)
    {
        $this->type = self::TYPE;
        return parent::beforeSave($insert);
    }
}
```

然后 `models/HeavyCar.php`:

```php
namespace app\models;

class HeavyCar extends Car
{
    const TYPE = 'heavy';

    public function init()
    {
        $this->type = self::TYPE;
        parent::init();
    }

    public static function find()
    {
        return new CarQuery(get_called_class(), ['type' => self::TYPE, 'tableName' => self::tableName()]);
    }

    public function beforeSave($insert)
    {
        $this->type = self::TYPE;
        return parent::beforeSave($insert);
    }
}
```

现在我们需要在 `Car`模型中覆盖 `instantiate` 方法：

```php
public static function instantiate($row)
{
    switch ($row['type']) {
        case SportCar::TYPE:
            return new SportCar();
        case HeavyCar::TYPE:
            return new HeavyCar();
        default:
           return new self;
    }
}
```

另外，我们需要在 `Car` 模型中覆盖 `tableName` 方法，以便所涉及到的模型使用同一个表；

```php
public static function tableName()
{
    return '{{%car%}}';
}
```

OK。我们来试试在`SiteController` 控制器中创建 `actionTest` 并运行它试试看：

```php
// 找到我们拥有的所有汽车
$cars = Car::find()->all();
foreach ($cars as $car) {
    echo "$car->id $car->name " . get_class($car) . "<br />";
}

// 查找任何 跑车
$sportCar = SportCar::find()->limit(1)->one();
echo "$sportCar->id $sportCar->name " . get_class($sportCar) . "<br />";
```

输出应该是：

```
1 Kamaz app\models\HeavyCar
2 Ferrari app\models\SportCar
3 BMW app\models\Car
2 Ferrari app\models\SportCar
```

这意味着模型现在根据 `type` 字段实例化，搜索按预期执行。

## 执行原理

`SportCar` 和 `HeavyCar` 模型非常相似。 他们都从 `Car` 延伸，并且有两种方法被覆盖。 在
`find` 方法，我们实例化一个存储汽车类型的自定义查询类并将其应用于在为数据库查询形成SQL之前调用的 `prepare` 方法。 `SportCar` 只会搜索跑车， `HeavyCar` 只会搜索重型车。 在 `beforeSave`中，我们确保在保存类时将正确的 `type` 写入数据库。为方便起见 `TYPE` 定义为常量。

 `Car` 模型除了 `instantiate` 方法之外几乎是Gii生成的,调用这个方法在数据库中检索数据并将要用于初始化类属性。 
返回值未初始化类实例和传递给该方法的唯一参数是从数据库检索的数据行。 正是我们需要的实现是一个简单的switch语句，我们检查`type`字段是否匹配我们所支持的类的类型。如果是，则返回该类的实例。 如果没有匹配，则返回到 `Car` 模型实例。

This method is called after data is retrieved from database and is about to be used to initialize class properties. 
Return value is uninitialized class instance and the only argument passed to the method is the row of data retrieved from the database. Exactly what we need.
The implementation is a simple switch statement where we're checking if the `type` field matches type of the classes we suport.
If so, an instance of the class is returned. If nothing matches, it falls back to returning a `Car` model instance. 

## 处理唯一值

If you have a column marked as unique, to prevent breaking the `UniqueValidator`` you need to specify the `targetClass`
property.

```php
    public function rules()
    {
        return [
            [['MyUniqueColumnName'], 'unique', 'targetClass' => '\app\models\Car'],
        ];
    }
```
