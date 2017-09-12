php代码审计中的重要函数，总结自《代码审计：企业级web代码安全架构》
### SQL注入
1. magic_quotes_gpc
2. addslashes
3. mysql_real_escape_string
4. intval

### XSS
1. print
2. pirnt_r
3. echo
4. printf
5. sprintf
6. die
7. var_dump
8. var_export

### 文件包含
1. include
2. include_once
3. require
4. require_once

### 文件读取
1. file_get_contents
2. highlight_file
3. fopen
4. readfile
5. fread
6. fget
7. fgets
8. parse_ini_file
9. show_source
10. file

### 文件上传
1. move_upload_file

### 代码执行
1. eval
2. assert
3. preg_replace
4. call_user_func
5. call_user_func_array
6. array_map

### 命令执行
1. system
2. exec
3. shell_exec
4. passthru
5. pcntl_exec
6. popen
7. proc_open

### 变量覆盖
1. ectract
2. parse_str
3. import_request_variables
4. $$

### mysql报错注入
1. floor
2. updatexml
3. extractvalue
4. GeometryCollection
5. polygon
6. multipoint
7. multilinestring
8. multipolygon
9. linestring
10. exp

### php安全编码
1. addslashes
2. mysql_real_escape_string
3. mysql_escape_string
4. str_replace
5. strops
6. htmlspecialchars
7. strip_args

### 防止命令注入
1. escapeshellcmd
2. escapeshellarg


