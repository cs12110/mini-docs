# Jdbc 转换数据字典

在项目中一般都要写好数据字典,但是每一个表都要自己写那些重复性的东西,所以打算使用 jdbc 来读取数据库的信息,然后转换成 md 文件,再由 md 的相关软件转换成 word 文档.

以下程序依赖 jar:`mysql连接驱动`,`fastjson`,请知悉.

---

## 1. 数据库连接工具类

用于连接数据库

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

/**
 * Jdbc工具类
 *
 * <p>
 *
 * @author cs12110 2018年3月30日上午8:33:25
 *
 */
public class JdbcUtil {

	/**
	 * 获取mysql数据库连接
	 *
	 * @param url
	 *            url连接地址
	 * @param user
	 *            用户
	 * @param password
	 *            密码
	 * @return {@link Connection}
	 */
	public static Connection getMySqlConn(String url, String user, String password) {
		String driver = "com.mysql.jdbc.Driver";
		return getConnection(driver, url, user, password);
	}

	/**
	 * 获取数据库连接
	 *
	 * @param driverName
	 *            驱动名称
	 * @param url
	 *            数据库连接url
	 * @param user
	 *            数据库连接用户
	 * @param password
	 *            用户登录密码
	 * @return {@link Connection}
	 */
	public static Connection getConnection(String driverName, String url, String user, String password) {
		Connection conn = null;
		try {
			Class.forName(driverName);
			conn = DriverManager.getConnection(url, user, password);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return conn;
	}

	/**
	 * 释放资源
	 *
	 * @param conn
	 *            连接
	 * @param stm
	 *            声明
	 * @param rs
	 *            结果集
	 */
	public static void fuckoff(Connection conn, Statement stm, ResultSet rs) {
		try {
			if (null != rs && !rs.isClosed()) {
				rs.close();
			}
		} catch (Exception e) {
			// do nothing
		}

		try {
			if (null != stm && !stm.isClosed()) {
				stm.close();
			}
		} catch (Exception e) {
			// do nothing
		}

		try {
			if (null != conn && !conn.isClosed()) {
				conn.close();
			}
		} catch (Exception e) {
			// do nothing
		}
	}
}
```

---

## 2. 构建表的列属性枚举

根据 metaData 获取表里面的属性,得到枚举

```java
/**
 * 列属性
 *
 * @author root
 *
 */
public enum ColumnMetaEnum {

	/**
	 * 数据库名称
	 */
	TABLE_CAT("TABLE_CAT"),

	/**
	 *
	 */
	TABLE_SCHEM("TABLE_SCHEM"),

	/**
	 * 表 名称
	 */
	TABLE_NAME("TABLE_NAME"),

	/**
	 * 列名称
	 */
	COLUMN_NAME("COLUMN_NAME"),

	/**
	 * 数据类型
	 */
	DATA_TYPE("DATA_TYPE"),

	/**
	 * 类型名称
	 */
	TYPE_NAME("TYPE_NAME"),

	/**
	 * 列
	 */
	COLUMN_SIZE("COLUMN_SIZE"),

	/**
	 * 缓冲区
	 */
	BUFFER_LENGTH("BUFFER_LENGTH"),

	/**
	 *
	 */
	DECIMAL_DIGITS("DECIMAL_DIGITS"),

	/**
	 *
	 */
	NUM_PREC_RADIX("NUM_PREC_RADIX"),

	/**
	 * 是否为空
	 */
	NULLABLE("NULLABLE"),

	/**
	 * 备注
	 */
	REMARKS("REMARKS"),

	/**
	 *
	 */
	COLUMN_DEF("COLUMN_DEF"),

	/**
	 *
	 */
	SQL_DATA_TYPE("SQL_DATA_TYPE"),

	/**
	 *
	 */
	SQL_DATETIME_SUB("SQL_DATETIME_SUB"),

	/**
	 *
	 */
	CHAR_OCTET_LENGTH("CHAR_OCTET_LENGTH"),

	/**
	 * 列号
	 */
	ORDINAL_POSITION("ORDINAL_POSITION"),

	/**
	 * 是否可以为空
	 */
	IS_NULLABLE("IS_NULLABLE"),

	/**
	 *
	 */
	SCOPE_CATALOG("SCOPE_CATALOG"),

	/**
	 *
	 */
	SCOPE_SCHEMA("SCOPE_SCHEMA"),

	/**
	 *
	 */
	SCOPE_TABLE("SCOPE_TABLE"),

	/**
	 *
	 */
	SOURCE_DATA_TYPE("SOURCE_DATA_TYPE"),

	/**
	 * 自动增长
	 */
	IS_AUTOINCREMENT("IS_AUTOINCREMENT"),

	/**
	 * 主键
	 */
	IS_GENERATEDCOLUMN("IS_GENERATEDCOLUMN");

	private final String mateName;

	private ColumnMetaEnum(String mateName) {
		this.mateName = mateName;
	}

	public String getMateName() {
		return mateName;
	}

}
```

---

## 3. 构建表信息实体类

构建相关实体类,映射数据库表信息.

```java
import com.alibaba.fastjson.JSON;

/**
 * 字段信息实体类
 *
 * @author root
 *
 */
public class FieldBean {

	/**
	 * 名称
	 */
	private String name;

	/**
	 * 类型
	 */
	private String type;

	/**
	 * 长度
	 */
	private String length;

	/**
	 * 是否为空
	 */
	private String nullable;

	/**
	 * 描述
	 */
	private String desc;

	/**
	 * 主键
	 */
	private String isKey;

	/**
	 * 自动增长
	 */
	private String autoIncre;

	/**
	 * 默认值
	 */
	private String defValue;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getType() {
		return type;
	}

	public void setType(String type) {
		this.type = type;
	}

	public String getLength() {
		return length;
	}

	public void setLength(String length) {
		this.length = length;
	}

	public String getNullable() {
		return nullable;
	}

	public void setNullable(String nullable) {
		this.nullable = nullable;
	}

	public String getDesc() {
		return desc;
	}

	public void setDesc(String desc) {
		this.desc = desc;
	}

	public String getIsKey() {
		return isKey;
	}

	public void setIsKey(String isKey) {
		this.isKey = isKey;
	}

	public String getAutoIncre() {
		return autoIncre;
	}

	public void setAutoIncre(String autoIncre) {
		this.autoIncre = autoIncre;
	}

	public String getDefValue() {
		return defValue;
	}

	public void setDefValue(String defValue) {
		this.defValue = defValue;
	}

	@Override
	public String toString() {
		return JSON.toJSONString(this);
	}
}
```

---

## 4. 编写数据库信息工具类

主要用于获取数据库相关信息

```java
import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import com.mysql.bean.FieldBean;
import com.mysql.enums.ColumnMetaEnum;

/**
 * 数据库信息工具类
 *
 * @author root
 *
 */
public class DbInfoUtil {

	/**
	 * 获取数据库里面所有的表
	 *
	 * @param conn
	 *            连接
	 * @param dbName
	 *            数据库名称
	 * @return List
	 */
	public static List<String> getTableList(Connection conn, String dbName) {
		List<String> tableList = new ArrayList<>();
		try {
			DatabaseMetaData dbMetaData = conn.getMetaData();
			ResultSet columns = dbMetaData.getColumns(dbName, null, null, null);
			Set<String> tableSet = new HashSet<String>();
			while (columns.next()) {
				tableSet.add(getMetaVal(columns, ColumnMetaEnum.TABLE_NAME));
			}
			tableList.addAll(tableSet);
			JdbcUtil.fuckoff(null, null, columns);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return tableList;
	}

	/**
	 * 获取表里面所有的列的属性
	 *
	 * @param conn
	 *            连接
	 * @param dbName
	 *            数据库名称
	 * @param tableName
	 *            表名称
	 * @return List
	 */
	public static List<FieldBean> getFieldFromTable(Connection conn, String dbName, String tableName) {
		List<FieldBean> fieldList = new ArrayList<FieldBean>();
		try {
			DatabaseMetaData dbMetaData = conn.getMetaData();
			ResultSet columns = dbMetaData.getColumns(dbName, "%", tableName, "%");
			String key = getPrimaryKey(dbMetaData, dbName, tableName);
			while (columns.next()) {
				FieldBean field = new FieldBean();
				field.setAutoIncre(columns.getString(ColumnMetaEnum.IS_AUTOINCREMENT.getMateName()));
				field.setName(getMetaVal(columns, ColumnMetaEnum.COLUMN_NAME));
				field.setType(getMetaVal(columns, ColumnMetaEnum.TYPE_NAME));
				field.setDesc(getMetaVal(columns, ColumnMetaEnum.REMARKS));
				field.setNullable(getMetaVal(columns, ColumnMetaEnum.IS_NULLABLE));
				field.setLength(getMetaVal(columns, ColumnMetaEnum.COLUMN_SIZE));
				field.setDefValue(getMetaVal(columns, ColumnMetaEnum.COLUMN_DEF));
				if (field.getName().equals(key)) {
					field.setIsKey("YES");
				}
				fieldList.add(field);
			}
			JdbcUtil.fuckoff(null, null, columns);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return fieldList;
	}

	/**
	 * 获取数据库表的主键
	 *
	 * @param dbMetaData
	 *            数据库元数据
	 * @param dbName
	 *            数据库名称
	 * @param tableName
	 *            表名称
	 * @return String
	 */
	private static String getPrimaryKey(DatabaseMetaData dbMetaData, String dbName, String tableName) {
		String key = "";
		try {
			ResultSet primaryKeys = dbMetaData.getPrimaryKeys(dbName, null, tableName);
			if (primaryKeys.next()) {
				key = getMetaVal(primaryKeys, ColumnMetaEnum.COLUMN_NAME);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return key;
	}

	/**
	 * 获取表里面的信息
	 *
	 * @param result
	 *            {@link ResultSet}
	 * @param meta
	 *            {@link ColumnMetaEnum}
	 * @return String
	 */
	private static String getMetaVal(ResultSet result, ColumnMetaEnum meta) {
		try {
			return result.getString(meta.getMateName());
		} catch (Exception e) {
			e.printStackTrace();
		}
		return "";
	}
}
```

---

## 5. markdown 内容工具类

主要用于生成 markdown 内容和生成 markdown 文件

```java
import java.io.FileOutputStream;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Set;

import cn.rojao.mysql.bean.FieldBean;

/**
 * Markdown文件生成类
 *
 * <p>
 *
 * @author root 2018年4月13日上午10:03:20
 *
 */
public class MarkdownUtil {

	/**
	 * 创建md文件
	 *
	 * @param path
	 *            路径
	 * @param infoMap
	 *            信息集合
	 */
	public static void buildMd(String path, Map<String, List<FieldBean>> infoMap) {
		StringBuilder md = new StringBuilder();

		md.append("# 数据库字典");
		md.append(System.lineSeparator());
		md.append("---");
		md.append(System.lineSeparator());
		md.append("本文档主要用于说明数据结构.");
		md.append(System.lineSeparator());

		Set<String> keys = infoMap.keySet();
		List<String> keyList = new ArrayList<>(keys);
		Collections.sort(keyList);

		for (String each : keyList) {
			List<FieldBean> fieldList = infoMap.get(each);
			md.append(buildEachTableMdContent(each, fieldList));
			md.append(System.lineSeparator());
			md.append(System.lineSeparator());
		}

		// infoMap.forEach((tableName, fieldList) -> {
		// md.append(buildEachTableMdContent(tableName, fieldList));
		// md.append(System.lineSeparator());
		// md.append(System.lineSeparator());
		// });

		write2File(path, md.toString());
	}

	/**
	 * 构建每一个数据表的markdown内容
	 *
	 * @param tableName
	 *            表名称
	 * @param list
	 *            属性列表
	 * @return String
	 */
	private static String buildEachTableMdContent(String tableName, List<FieldBean> list) {
		StringBuilder table = new StringBuilder();

		table.append("### ").append(tableName);
		table.append(System.lineSeparator());
		table.append("---");
		table.append(System.lineSeparator());
		table.append(System.lineSeparator());
		table.append("表名称: `").append(tableName).append("`");
		table.append(System.lineSeparator());
		table.append("中文描述: ").append("");
		table.append(System.lineSeparator());
		table.append(System.lineSeparator());

		table.append("|字段名称|字段类型|自增|非空|默认值|主键|备注|");
		table.append(System.lineSeparator());
		table.append("|---|---|---|:---:|---|:---:|---|");
		table.append(System.lineSeparator());
		for (int index = 0, size = list.size(); index < size; index++) {
			FieldBean bean = list.get(index);
			table.append("|");
			table.append(bean.getName()).append("|");
			table.append(bean.getType().toLowerCase()).append("[").append(bean.getLength()).append("]").append("|");
			table.append(fixYes(bean.getAutoIncre())).append("|");
			table.append(fixNotNo(bean.getNullable())).append("|");
			table.append(fixNull(bean.getDefValue())).append("|");
			table.append(fixYes(bean.getIsKey())).append("|");
			table.append(bean.getDesc()).append("|");
			table.append(System.lineSeparator());
		}
		return table.toString();
	}

	/**
	 * 修复空显示
	 *
	 * @param str
	 *            字符串
	 * @return String
	 */
	private static String fixNull(String str) {
		return null == str || "null".equals(str) ? "" : str;
	}

	/**
	 *
	 * 是则返回y,否则返回""
	 *
	 * @param val
	 *            值
	 * @return String
	 */
	private static String fixYes(String val) {
		return "YES".equals(val) ? "√" : "";
	}

	/**
	 * 不为NO,是则返回y,否则返回""
	 *
	 * @param val
	 *            值
	 * @return String
	 */
	private static String fixNotNo(String val) {
		return "NO".equals(val) ? "√" : "";
	}

	/**
	 * 内容写入文件
	 *
	 * @param path
	 *            文件路径
	 * @param content
	 *            文件内容
	 */
	private static void write2File(String path, String content) {
		try {
			FileOutputStream fout = new FileOutputStream(path);
			fout.write(content.getBytes());
			fout.flush();
			fout.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

---

## 6. App

整合整一个流程.

```java
import java.sql.Connection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.mysql.bean.FieldBean;
import com.mysql.util.DbInfoUtil;
import com.mysql.util.JdbcUtil;
import com.mysql.util.MarkdownUtil;

/**
 * 制造markdown文件
 *
 * @author root
 *
 */
public class App {

	public static void main(String[] args) {
		String dbUrl = "jdbc:mysql://47.98.104.252:3306";
		String dbUser = "root";
		String dbPassword = "Root@3306";
		String dbName = "forget_it";
		String mdPath = "d://forget-it.md";

		System.out.println("------ 开始 -------");
		execute(dbUrl, dbUser, dbPassword, dbName, mdPath);
		System.out.println("------ 结束 -------");
	}

	/**
	 * 创建md文件
	 *
	 * @param url
	 *            数据库地址
	 * @param user
	 *            用户
	 * @param password
	 *            密码
	 * @param dbName
	 *            数据库名称
	 * @param mdPath
	 *            markdown文件保存路径
	 */
	private static void execute(String url, String user, String password, String dbName, String mdPath) {
		Connection conn = JdbcUtil.getMySqlConn(url, user, password);
		List<String> tableList = DbInfoUtil.getTableList(conn, dbName);
		Map<String, List<FieldBean>> infoMap = new HashMap<>();

		for (String tableName : tableList) {
			List<FieldBean> fieldList = DbInfoUtil.getFieldFromTable(conn, dbName, tableName);
			infoMap.put(tableName, fieldList);
		}
		JdbcUtil.fuckoff(conn, null, null);
		MarkdownUtil.buildMd(mdPath, infoMap);
	}
}
```
