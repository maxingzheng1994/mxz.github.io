##### 基本用法
```puml
Bob -> Alice : Hello
Alice -> Bob: Go
```
##### 设置不同的角色
```puml

actor Foo1
boundary Foo2
control Foo3
entity Foo4
database Foo5
collections Foo6
Foo1 -> Foo2 : To boundary
Foo1 -> Foo3 : To control
Foo1 -> Foo4 : To entity
Foo1 -> Foo5 : To database
Foo1 -> Foo6 : To collections

```

##### 不用的箭头样式
```puml
Bob ->x Alice
Bob -> Alice
Bob ->> Alice
Bob -\ Alice
Bob \\- Alice
Bob //-- Alice

Bob ->o Alice
Bob o\\-- Alice

Bob <-> Alice
Bob <->o Alice
Bob -[#red]> Alice : hello
Alice -[#0000FF]->Bob : ok
```

##### 分段 分页

```puml
== Initialization ==

Alice -> Bob: Authentication Request
Bob --> Alice: Authentication Response

newpage A title for the\nlast page

Alice -> Bob: Another authentication Request
Alice <-- Bob: another authentication Response

```

##### 生命线

```puml
participant User

User -> A: DoWork
activate A #FFBBBB

A -> A: Internal call
activate A #DarkSalmon

A -> B: << createRequest >>
activate B

B --> A: RequestCreated
deactivate B
deactivate A
A -> User: Done
deactivate A
```
##### 图例注脚等
```puml
header Page Header
footer Page %page% of %lastpage%

title Example Title

Alice -> Bob : message 1
note left: this is a first note

Alice -> Bob : message 2
```
C4架构图
```puml
!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Context.puml
!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Container.puml
!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Component.puml

System(systemAlias, "System", "这可以看作系统上下文(Context)")
Container(containerAlias, "Container", "这是Container")
Person(personAlias, "Person", "这可以看作是组件(Component)")

Rel(personAlias, containerAlias, "Label", "设置关联关系")
```

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```