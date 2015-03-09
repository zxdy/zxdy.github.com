记一个写pl/sql比较有用的技巧。当oracle的存储过程运行出现异常的时候，虽然可以被exception抓到，但是无法定位究竟是在之前的业务处理逻辑代码哪一行出现了问题，调试起来不方便。比如下面的demo运行之后

```pl/sql
CREATE OR REPLACE
  FUNCTION FUNC
    RETURN NUMBER
  AS
    v_in NUMBER;
    v_out NUMBER;

  BEGIN
    v_in :=12;
    dbms_output.put_line(v_in);
    v_out:=v_in/0;
    RETURN v_out;
  EXCEPTION
  WHEN OTHERS THEN
    dbms_output.put_line('######exception:');
  END FUNC;
```

显示的结果如下，并没有显示真正异常的代码行号

```text
Connecting to the database dev74.
ORA-06503: PL/SQL: Function returned without value
ORA-06512: at "TLSPID.FUNC", line 16
ORA-06512: at line 5
12
######exception:
Process exited.
Disconnecting from the database dev74.
```

但是只要在exception代码块中加上第16行的代码

```pl/sql
CREATE OR REPLACE
  FUNCTION FUNC
    RETURN NUMBER
  AS
    v_in NUMBER;
    v_out NUMBER;

  BEGIN
    v_in :=12;
    dbms_output.put_line(v_in);
    v_out:=v_in/0;
    RETURN v_out;
  EXCEPTION
  WHEN OTHERS THEN
    dbms_output.put_line('######exception:');
    dbms_output.put_line(dbms_utility.format_error_backtrace());
  END FUNC;
```

运行后结果如下，可以看到第7行显示了出错的地方。实际使用的时候注意出错的行号应该是报错的下一行。比如下面显示第10行报错，但实际应该是第11行出错。

```text
Connecting to the database dev74.
ORA-06503: PL/SQL: Function returned without value
ORA-06512: at "TLSPID.FUNC", line 16
ORA-06512: at line 5
12
######exception:
ORA-06512: at "TLSPID.FUNC", line 10
Process exited.
Disconnecting from the database dev74.
```