在sql developer中，如果直接对pipelined语句进行断点debug会报错，那么怎样可以解决这个问题呢？可以用procedure包住这个函数，再进行单步调试。下面是演示的demo。  

##  准备pipelined

```pl/sql
CREATE TYPE t_tf_row AS OBJECT (
  id           NUMBER,
  description  VARCHAR2(50)
);
/

CREATE TYPE t_tf_tab IS TABLE OF t_tf_row;
/

CREATE OR REPLACE FUNCTION get_tab_ptf (p_rows IN NUMBER) RETURN t_tf_tab PIPELINED AS
BEGIN
  FOR i IN 1 .. p_rows LOOP
    PIPE ROW(t_tf_row(i, 'Description for ' || i));   
  END LOOP;

  RETURN;
END;
/
```

## 然后用procedure包住这个get_tab_ptf函数

```pl/sql
CREATE OR REPLACE
PROCEDURE test_pipeline
IS
BEGIN
  FOR cur_rec IN (SELECT *  FROM  TABLE(get_tab_ptf(10)))
  LOOP
    dbms_output.put_line('get one row!');
  END LOOP;
END;
```

## 断点位置

最后将断点打在test_pipeline的for循环上，对这个procedure进行单步调试就可以跳转到函数get_tab_ptf内部了。
