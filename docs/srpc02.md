## client

```cpp
// src/rpc_client.h
template<class RPCTYPE>
class RPCClient
{
public:
	using TASK = RPCClientTask<typename RPCTYPE::REQ, typename RPCTYPE::RESP>;

protected:
	using COMPLEXTASK = WFComplexClientTask<typename RPCTYPE::REQ,
											typename RPCTYPE::RESP>;

public:
	RPCClient(const std::string& service_name);
	virtual ~RPCClient() { };

	const RPCTaskParams *get_task_params() const;
	const std::string& get_service_name() const;

	void task_init(COMPLEXTASK *task) const;

	void set_keep_alive(int timeout);
	void set_watch_timeout(int timeout);
	void add_filter(RPCFilter *filter);

protected:
	template<class OUTPUT>
	TASK *create_rpc_client_task(const std::string& method_name,
								 std::function<void (OUTPUT *, RPCContext *)>&& done)
	{
		std::list<RPCModule *> module;
		for (int i = 0; i < SRPC_MODULE_MAX; i++)
		{
			if (this->modules[i])
				module.push_back(this->modules[i]);
		}

		auto *task = new TASK(this->service_name,
							  method_name,
							  &this->params.task_params,
							  std::move(module),
							  [done](int status_code, RPCWorker& worker) -> int {
				return ClientRPCDoneImpl(status_code, worker, done);
			});

		this->task_init(task);

		return task;
	}

	void init(const RPCClientParams *params);
	std::string service_name;

private:
	void __task_init(COMPLEXTASK *task) const;

protected:
	RPCClientParams params;
	ParsedURI uri;

private:
	struct sockaddr_storage ss;
	socklen_t ss_len;
	bool has_addr_info;
	std::mutex mutex;
	RPCModule *modules[SRPC_MODULE_MAX] = { 0 };
};
```

这里有几个比较重要的部件

1. task 

2. RPCModule

## task 

在workflow中，是以task为单位，去进行串并联进行任务的，每一次rpc调用其实也是进行一次网络收发(todo, 此处详细描述下过程)

所以我们需要创建我们独有协议的task

```cpp
using TASK = RPCClientTask<typename RPCTYPE::REQ, typename RPCTYPE::RESP>;
```

```cpp
// src/rpc_task.inl
template<class RPCREQ, class RPCRESP>
class RPCClientTask : public WFComplexClientTask<RPCREQ, RPCRESP>
{
public:
	// before rpc call
	void set_data_type(RPCDataType type);
	void set_compress_type(RPCCompressType type);
	void set_retry_max(int retry_max);
	void set_attachment_nocopy(const char *attachment, size_t len);
	int set_uri_fragment(const std::string& fragment);
	int serialize_input(const ProtobufIDLMessage *in);
	int serialize_input(const ThriftIDLMessage *in);

	// similar to opentracing: log({{"event", "error"}, {"message", "application log"}});
	void log(const RPCLogVector& fields);

	// Baggage Items, which are just key:value pairs that cross process boundaries
	void add_baggage(const std::string& key, const std::string& value);
	bool get_baggage(const std::string& key, std::string& value);

	bool set_http_header(const std::string& name, const std::string& value);
	bool add_http_header(const std::string& name, const std::string& value);

	// JsonPrintOptions
	void set_json_add_whitespace(bool on);
	void set_json_always_print_enums_as_ints(bool on);
	void set_json_preserve_proto_field_names(bool on);
	void set_json_always_print_primitive_fields(bool on);

protected:
	using user_done_t = std::function<int (int, RPCWorker&)>;

	using WFComplexClientTask<RPCREQ, RPCRESP>::set_callback;

	void init_failed() override;
	bool check_request() override;
	CommMessageOut *message_out() override;
	bool finish_once() override;
	void rpc_callback(WFNetworkTask<RPCREQ, RPCRESP> *task);
	int first_timeout() override { return watch_timeout_; }

public:
	RPCClientTask(const std::string& service_name,
				  const std::string& method_name,
				  const RPCTaskParams *params,
				  const std::list<RPCModule *>&& modules,
				  user_done_t&& user_done);

	bool get_remote(std::string& ip, unsigned short *port) const;

	RPCModuleData *mutable_module_data() { return &module_data_; }
	void set_module_data(RPCModuleData data) { module_data_ = std::move(data); }

private:
	template<class IDL>
	int __serialize_input(const IDL *in);

	user_done_t user_done_;
	bool init_failed_;
	int watch_timeout_;

	RPCModuleData module_data_;
	std::list<RPCModule *> modules_;
};
```
