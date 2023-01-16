## 1：外键

子表（外键所在的表）

```kotlin
/**
 * 子表的外键需要开启“索引”，否则Room会发出Warning。
 *
 * ForeignKey可以设置是否deferred(推迟)，默认为false，如果设置为true，则在出现不一致冲突时不会立即报错，而会在一个
 * transaction(事务)结束之后才报错。在本例中由于没有“显式”定义事务，所以这个字段是否为true表现一样——在冲突发生时报错。
 */
@Entity(tableName = "report_card",
    foreignKeys = [ForeignKey(
        entity = Type::class,
        childColumns = ["type_name"],
        parentColumns = ["type_name"],
        onDelete = ForeignKey.CASCADE,// 删除类型的时候，级联删除记录
        onUpdate = ForeignKey.CASCADE// 更改类型的时候，级联更改
    )])
data class Record(
    @PrimaryKey(autoGenerate = true)@ColumnInfo(name = "record_id") val id: Int = 0,
    @ColumnInfo(name = "time_stamp") val timeStamp: Long,
    @ColumnInfo(name = "number") val number: Double,
    @ColumnInfo(name = "type_name", index = true) val typeName: String,
    @ColumnInfo(name = "remark") val remark: String,
)
```

父表

```kotlin
@Entity(tableName = "type")
data class Type(
    @PrimaryKey @ColumnInfo(name = "type_name") val name: String,
    @ColumnInfo(name = "icon_id") val iconId: Int,
    @ColumnInfo(name = "order") val order: Int,
    @ColumnInfo(name = "expense_or_income") val expenseOrIncome: ExpenseOrIncome,
    @ColumnInfo(name = "is_customise") val isCustomise: Boolean,
    @ColumnInfo(name = "should_show") val shouldShow: Boolean,
)
```



1. 当对子表操作（插入，更新）导致在父表中没有找到外键对应的情况，即参照出现不一致，就报错。
2. 当对父表操作（删除或更新），可以在onDelete和onUpdate指定要进行什么操作。这里两个都设置为级联。