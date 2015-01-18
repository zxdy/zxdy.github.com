有没有碰到过这种情况，当你的java程序连接oracle运行存储过程的时候，java控制台只是仅仅输出java程序的相关信息，
而存储过程中的dbms_output内容却没有办法显示？这样只能跑去sql developer 单独调试存储过程，而不能直接在eclipse
进行集成测试，出了问题也不好定位。  

所以在此分享一个解决方案： [原文链接][1] 。05年的一个方法，不知现在有没有更好的：）  

## 存储过程demo

```pl/sql
create or replace
PROCEDURE test_java_dbmsOutPut
IS
BEGIN
   dbms_output.put_line('im in store procedure');
END;
/
```

## 用来输出dbms_output 的类

```java
import java.sql.*;
class DbmsOutput 
 {
    /*
     * our instance variables. It is always best to use callable or prepared
     * statements and prepare (parse) them once per program execution, rather
     * then one per execution in the program. The cost of reparsing is very
     * high. Also -- make sure to use BIND VARIABLES!
     * 
     * we use three statments in this class. One to enable dbms_output -
     * equivalent to SET SERVEROUTPUT on in SQL*PLUS. another to disable it --
     * like SET SERVEROUTPUT OFF. the last is to "dump" or display the results
     * from dbms_output using system.out
     */
    private CallableStatement enable_stmt;
    
    private CallableStatement disable_stmt;
    
    private CallableStatement show_stmt;
    
    /*
     * our constructor simply prepares the three statements we plan on
     * executing.
     * 
     * the statement we prepare for SHOW is a block of code to return a String
     * of dbms_output output. Normally, you might bind to a PLSQL table type but
     * the jdbc drivers don't support PLSQL table types -- hence we get the
     * output and concatenate it into a string. We will retrieve at least one
     * line of output -- so we may exceed your MAXBYTES parameter below. If you
     * set MAXBYTES to 10 and the first line is 100 bytes long, you will get the
     * 100 bytes. MAXBYTES will stop us from getting yet another line but it
     * will not chunk up a line.
     */
    public DbmsOutput(Connection conn)
        throws SQLException {
        enable_stmt = conn.prepareCall("begin dbms_output.enable(:1); end;");
        disable_stmt = conn.prepareCall("begin dbms_output.disable; end;");
        show_stmt =
            conn.prepareCall("declare "
                + "    l_line varchar2(255); "
                + "    l_done number; "
                + "    l_buffer long; "
                + "begin "
                + "  loop "
                + "    exit when length(l_buffer)+255 > :maxbytes OR l_done = 1; "
                + "    dbms_output.get_line( l_line, l_done ); "
                + "    l_buffer := l_buffer || l_line || chr(10); "
                + "  end loop; " + " :done := l_done; "
                + " :buffer := l_buffer; " + "end;");
    }
    
    /*
     * enable simply sets your size and executes the dbms_output.enable call
     */
    public void enable(int size)
        throws SQLException {
        enable_stmt.setInt(1, size);
        enable_stmt.executeUpdate();
    }
    
    /*
     * disable only has to execute the dbms_output.disable call
     */
    public void disable()
        throws SQLException {
        disable_stmt.executeUpdate();
    }
    
    /*
     * show does most of the work. It loops over all of the dbms_output data,
     * fetching it in this case 32,000 bytes at a time (give or take 255 bytes).
     * It will print this output on stdout by default (just reset what
     * System.out is to change or redirect this output).
     */
    public void show()
        throws SQLException {
        int done = 0;
        show_stmt.registerOutParameter(2, java.sql.Types.INTEGER);
        show_stmt.registerOutParameter(3, java.sql.Types.VARCHAR);
        for (;;) {
            show_stmt.setInt(1, 32000);
            show_stmt.executeUpdate();
            System.out.print(show_stmt.getString(3));
            if ((done = show_stmt.getInt(2)) == 1)
                break;
        }
    }
    
    /*
     * close closes the callable statements associated with the DbmsOutput
     * class. Call this if you allocate a DbmsOutput statement on the stack and
     * it is going to go out of scope -- just as you would with any callable
     * statement, result set and so on.
     */
    public void close()
        throws SQLException {
        enable_stmt.close();
        disable_stmt.close();
        show_stmt.close();
    }
}
```

## 测试代码

```java

import java.sql.*;
public class Test{
    public static void main(String args[])
        throws SQLException {
        DriverManager.registerDriver(new oracle.jdbc.driver.OracleDriver());
        Connection conn = DBAgent.getConn(); 
        conn.setAutoCommit(false);
        Statement stmt = conn.createStatement();
        DbmsOutput dbmsOutput = new DbmsOutput(conn);
        dbmsOutput.enable(1000000);
        stmt.execute("begin test_java_dbmsOutPut; end;"); #调用存储过程
        stmt.close();
        dbmsOutput.show(); #显示DBMS_OUTPUT
        dbmsOutput.close();
        conn.close();
    }   
}

```

## 输出的结果  

运行测试代码之后，就可以看见在java 控制台输出了: **im in store procedure**


> 这样就可以让java为我们输出DBMS_OUTPUT，就像SQL*PLUS一样。你需要做的仅仅是在java运行存储过程之后，调用DbmsOutput.show() ，就能显示这个存储过程内的DBMS_OUTPUT，是不是很方便？再也不用在eclipse和sql develper之间两头调试了

[1]: http://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:45027262935845