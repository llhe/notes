leveldb cache
==================
leveldb的cache分为TableCache和BlockCache:
  * TableCache，主要缓存了文件句柄，索引块，bloom filter

  table cache 存取的key类型为`TableAndFile`:
  ```C++
  struct TableAndFile {
	RandomAccessFile* file; // 打开的文件句柄
	Table* table; // Table对象，包含index数据和bloom filter数据等
  };

  ```

  Table中仅有一项Rep结构，采用pimpl范式隐藏实现(effective c++):
  ```C++
  class Table {
    // ...
  private:
    struct Rep;
    Rep *rep_;
    // ...
  };
  ```
  
  Rep结构定义：
  ```C++
  struct Table::Rep {
	~Rep() {
	  delete filter;
	  delete [] filter_data;
	  delete index_block;
	}

	Options options;
	Status status;
	RandomAccessFile* file;
	uint64_t cache_id;
	FilterBlockReader* filter;
	const char* filter_data;

	BlockHandle metaindex_handle;  // Handle to metaindex_block: saved from footer
	Block* index_block;
  };
  ```

  * BlockCache: 数据块

