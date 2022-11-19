## generator工具

generator工具源码在src/generator/ 下

使用方式: 

```
srpc_generator -i <IDL_FILE> -o <OUTPUT_DIR>
srpc_generator --input <IDL_FILE> --output <OUTPUT_DIR>
```

先找到入门main函数

```cpp
if (argc == 2 &&
	(strcmp(argv[1], "-v") == 0 || strcmp(argv[1], "--version") == 0))
{
	fprintf(stderr, "srpc_generator version %s\n", SRPC_VERSION);
	return 0;
}
else if (argc == 3)
{
	idl_type = check_idl_type(argv[1]);
	if (idl_type == TYPE_UNKNOWN)
	{
		fprintf(stderr, "ERROR: Invalid IDL file \"%s\"\n", argv[1]);
		fprintf(stderr, SRPC_GENERATOR_USAGE, argv[0]);
		return 0;
	}
}
else if (argc == 4)
{
	if (strcasecmp(argv[1], "protobuf") == 0)
		idl_type = TYPE_PROTOBUF;
	else if (strcasecmp(argv[1], "thrift") == 0)
		idl_type = TYPE_THRIFT;
	else
	{
		fprintf(stderr, "ERROR: Invalid IDL type \"%s\"\n", argv[1]);
		fprintf(stderr, SRPC_GENERATOR_USAGE, argv[0]);
		return 0;
	}

	idl_file_id++;
}
else
{
	fprintf(stderr, SRPC_GENERATOR_USAGE, argv[0]);
	return 0;
}
```

先是进行参数判断

如果没有输入是什么类型，我们会根据他的拓展来判断

```cpp
int check_idl_type(const char *filename)
{
	size_t len = strlen(filename);

	if (len > 6 && strcmp(filename + len - 6, ".proto") == 0)
		return TYPE_PROTOBUF;
	else if (len > 7 && strcmp(filename + len - 7, ".thrift") == 0)
		return TYPE_THRIFT;

	return TYPE_UNKNOWN;
}
```

然后我们开始构造generator开始生成文件

```cpp
const char *idl_file = argv[idl_file_id];
struct GeneratorParams params;
Generator gen(idl_type == TYPE_THRIFT ? true : false);

params.out_dir = argv[idl_file_id + 1];

char file_name[DIRECTORY_LENGTH] = { 0 };
size_t pos = 0;

if (idl_file[0] != '/' && idl_file[1] != ':')
{
	getcwd(file_name, DIRECTORY_LENGTH);
	pos = strlen(file_name) + 1;
	file_name[strlen(file_name)] = '/';
}

if (strlen(idl_file) >= 2 && idl_file[0] == '.' && idl_file[1] == '/')
	idl_file += 2;

snprintf(file_name + pos, DIRECTORY_LENGTH - pos, "%s", idl_file);

gen.generate(file_name, params);
```

他需要的参数就是生成的路径

```cpp
struct GeneratorParams
{
	const char *out_dir;
	const char *out_file;
};
```

