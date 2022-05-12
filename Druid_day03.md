# Druid解析SQL

## 抽象语法树AST
抽象语法树(AbstractSyntaxTree，AST)，或简称语法树(Syntax tree)，是源代码语法结构的一种抽象表示，Druid解析SQL也一样，会遵循一定的规则将SQL分析并构建成语法树AST。
Parser主要的作用是生成AST，Parser主要有两部分组成：词法分析、语法分析。
Druid收到一条SQL后，比如select a, b , from userTable where user_id =10，需要解析出每一个单词，并记录单词位置等，词法解析阶段，并不需要理解这条SQL的含义，专业术语Lexer。
词法分析完成后就是语法分析，作用就是要明确SQL的含义，语法分析的结果就是要明确这个单词的含义。
AST仅仅是语义的表示，但如何对这个语义进行表达，便需要去访问这棵AST，看它到底表达什么含义。通常遍历语法树，使用VISITOR模式去遍历，从根节点开始遍历，一直到最后一个叶子节点，在遍历的过程中，便不断地收集信息到一个上下文中，整个遍历过程完成后，对这棵树所表达的语法含义，已经被保存到上下文了。
通过Parser生成完整的AST抽象语法树。
## Druid SQL中AST节点类型
常用的AST节点主要有三种类型：SQLObject、SQLExpr、SQLStatement，其中最常使用的就莫过于SQLStatement ，其子类常见的有DruidSelectStatement、DruidInsertStatement、DruidUpdateStatement等
```
public interface SQLStatement extends SQLObject {}
public interface SQLObject {}
public interface SQLExpr extends SQLObject, Cloneable {}
```
### 常见的SQLStatement
最常用的Statement是SELECT/UPDATE/DELETE/INSERT
### 常见的SQLTableSource
常见的SQLTableSource包括SQLExprTableSource、SQLJoinTableSource、SQLSubqueryTableSource、SQLWithSubqueryClause.Entry
### SQLSelect & SQLSelectQuery
SQLSelectStatement包含一个SQLSelect，SQLSelect包含一个SQLSelectQuery，都是组成的关系。SQLSelectQuery有主要的两个派生类，分别是SQLSelectQueryBlock和SQLUnionQuery。
### SQLCreateTableStatement
建表语句包含了一系列方法，用于方便各种操作


## 流程分析

### SQLObject 对象
SQLObject对象是Druid体系中的顶层接口，用于描述所有SQL有关的对象，比如：ASTSQLObject的一个继承类。对应访问者模式中的Element，用于接收一个访问者对象(Visitor 的具体实现类)。

访问者遍历AST语法树

遍历AST树:
```
        sqlStatement.accept(visitor);
```
其中:
druid-master\src\main\java\com\alibaba\druid\sql\ast\SQLObjectImpl.java
```
    public SQLObjectImpl(){
    }

    public final void accept(SQLASTVisitor visitor) {
        if (visitor == null) {
            throw new IllegalArgumentException();
        }

        visitor.preVisit(this);

        accept0(visitor);

        visitor.postVisit(this);
    }

    protected abstract void accept0(SQLASTVisitor v);
	
```

accept0的具体实现:
druid-master\src\main\java\com\alibaba\druid\sql\dialect\mysql\ast\statement\MySqlUpdateStatement.java
```
    public void accept0(MySqlASTVisitor visitor) {
        if (visitor.visit(this)) {
            if (tableSource != null) {
                tableSource.accept(visitor);
            }

            if (from != null) {
                from.accept(visitor);
            }

            if (items != null) {
                for (int i = 0; i < items.size(); i++) {
                    SQLUpdateSetItem item = items.get(i);
                    if (item != null) {
                        item.accept(visitor);
                    }
                }
            }

            if (where != null) {
                where.accept(visitor);
            }

            if (orderBy != null) {
                orderBy.accept(visitor);
            }

            if (limit != null) {
                limit.accept(visitor);
            }


            if (hints != null) {
                for (int i = 0; i < hints.size(); i++) {
                    SQLCommentHint hint = hints.get(i);
                    if (hint != null) {
                        hint.accept(visitor);
                    }
                }
            }
        }
        visitor.endVisit(this);
    }
```
调用visitor.visit(this)方法，把对象自己传进去。visitor同时实现了visitor.visit(MySqlUpdateStatement mySqlUpdateStatement)方法，此时就会进入到visitor的实现里面去。

visitor访问完自身节点后，然后再去访问子节点，就如MySqlUpdateStatement实例一样。
```
tableSource.accept(visitor);
from.accept(visitor);
item.accept(visitor);
where.accept(visitor);
orderBy.accept(visitor);
...

```
### 自定义visitor
如果我们需要自定义visitor，那么必须实现SQLASTVisitor接口，该接口是抽象访问者，定义访问者的行为规范。SQLASTVisitorAdapter类已经帮我们实现了SQLASTVisitor，所以需要自定义visitor时，可以选择继承SQLASTVisitorAdapter，然后重写特定接口即可。比如：需要通过自定义visitor获取SQL的limit RowCount数量，可以以如下方式定义visitor，当使用该visitor遍历AST树时，如果遍历到SQLLimit节点，就会调用该重写方法。
// 自定义访问者
```
    class SQLCustomedVisitor extends SQLASTVisitorAdapter {

        protected boolean hasLimit = false;

        @Override
        public boolean visit(SQLLimit x) {
            System.out.println(x.getRowCount());
            hasLimit = true;
            return false;
        }

        public boolean isHasLimit() {
            return hasLimit;
        }
    }
```
