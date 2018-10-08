# How To Make Your Website Faster

## About
	- This page have a purpose to discussion about an issue probably helpful with everyone are web developer
	- I don't teach, no bet on it about anything or something

---

## Situation
	- Will be hosted on a website that has a source code.
	- Your work in limited time and must tune that site achieves the highest thoughput
	- You can use any ways e.g change structure database, paste index, add middleware, refartoring logic of application...
	- However It must ensure:
		1. Test of benchmark to rate point is correction
		2. Not change spec of host(Ram or CPU)
---

## 3 Technical skill
	- Knowledge about website running, the stack, how about website run.
	- Devops, tuning middleware layer integrate to be used nginx, mysql...
	- Using tool to benchmarking, detect bottleneck
	- familar with language programing of this source
---


## Content
### Database
	- Basicaly mysql or orther database system will support the shortest path to find out your data necessary.
	- However, not always mysql has find out the shortest path, must need support from developer and add index is a way.
#### Mysql query tuning
	- How to add indices is good?

	---

		- Mysql use B-Tree to save index (Storage engine is InnoDB)
		- ![Mysql_B-Tree](https://github.com/mui-le/blog/blob/master/mysql_b_tree.jpg)

		- If you don't know, you only know the point below:

			- It's tree
			- It's support range query
			- In order to support range query, It has the pointer between the leaf (instead of the conventional from parent to children).

	- Order for add index is very important!

	---

		- ~~you can see above that the pointer of the leaf has order from left to right. So your index was correspond~~
		- Example:
			```sql
			SELECT name, gender Quang FROM dig WHERE name = 'Diep' AND age = '30' AND language = 'japan' 
			```
			~~for example above, we need add an index cover for 3 fields to support mysql find quickly your data~~ 
			```sql
			ALTER TABLE dig ADD INDEX idx_name_age_language(name, age, language)
			```
			~~However multicolumn index will work when query select reverse or forward order with index inited, what happend when we change order of query~~
			```sql
				SELECT name, bar FROM dig WHERE name = 'Diep' AND language = 'vietnam' AND age = '30'
			```
			~~Mysql will don't know need reorder query and use `idx_name_age_language` to find. So you will need re-init index alert:
			```sql
				ALTER TABLE dig ADD INDEX idx_name_language_age(name, language, age)
			```
		* Note: the knowledge above is no longer true because mysql has support for now [index-condition-pushdown-optimization](https://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html)
		- Order of index will most likely be affected by the command `LIKE`. You can reference [multiple-column-indexes](https://dev.mysql.com/doc/refman/5.7/en/multiple-column-indexes.html)
			- Consider the two indexes:
			```sql
				create index idx_lf on name(last_name, first_name);
				create index idx_fl on name(first_name, last_name);
			```
			- Both of these should work equally well on:
			```sql
				where last_name = XXX and first_name = YYY
			```
			- idx_lf will be optimal for the following conditions:
			```sql
				where last_name = XXX
				where last_name like 'X%'
				where last_name = XXX and first_name like 'Y%'
				where last_name = XXX order by first_name
			```
			- idx_fl will be optimal for the following:
			```sql
				where first_name = YYY
				where first_name like 'Y%'
				where first_name = YYY and last_name like 'X%'
				where first_name = XXX order by last_name
			```
			- For many of these cases, both indexes could possibly be used, but one is optimal. For instance, consider idx_lf with the query:
			```sql
				where first_name = XXX order by last_name
			```
			- MySQL could read the entire table using idx_lf and then do the filtering after the order by. I don't think this is an optimization option in practice (for MySQL), but that can happen in other databases.

	- How about index will be affected with `OR` query?

	---

		- With `AND` query maybe everthing is pretty straight forward. But things are not so easy with `OR` query
			```sql
				SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;
			```
		- The other biggest betwwen `AND` query with `OR` query. that with `AND` query you can use `multi column index` but `OR` is not. Why?
		- The reason for above that is you need information index of field1, field2... But Do not need the in the same time because it's `OR` condition. So what is you solution in this case?

		- Mysql Has support index merge for this case So we could add an index for field1, an index for field2....an index for fieldm. Mysql will support them merge them, the first it will be find whose record relate index of field1, after that will be find the index relate of field2...then merge result of them.
		- [Document to reference](https://dev.mysql.com/doc/refman/5.7/en/index-merge-optimization.html) 

	- Covering index

	---

		- Covering index is type special of index. It contains all the data always needs to search
		- So we try to select the data that these index keys on where clause
			```sql
			SELECT name, bar FROM dig WHERE name = 'Diep' AND language = 'vietnam' AND age = '30'
			```
		- [Document reference](https://planet.mysql.com/entry/?id=661727)

	- Query function

	---

		- Let's careful with query function because mysql maybe not understand your query to use indexes
		- Example:
			```sql
				SELECT field FROM table WHERE field1 + 1 = 5
			```
			Index not work because mysql not enough smart to know that
			```sql
				SELECT field FROM table WHERE field = 4
			```

	- Should be use auto increment for primary key

	---

		- In other word, we can see insert into between sorted data-structure và non-sorted data-structure
		- of course, non-sorted data-structure will faster

	- Null or not null

	---

		- Mysql  support for null value, that meant exist state data non value.
		- So we shouldn't use null value for all field

	- Config buffer pool

	---

		- InnoDB use the memory is `buffer pool` for cache data & save index, this memory save by unit is `page` has volumn (default 16kb) and use LRU algorithm to evict cache.
		- Documents:
			- [The InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html)
			- [how large should be mysql innodb](https://dba.stackexchange.com/questions/27328/how-large-should-be-mysql-innodb-buffer-pool-size)
			- [glossary](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_page_size)
### System
	- I choose nginx for webserver (same Ito-san in confluence tech of him)
	- You can say some helpful of nginx:
		- Reverse proxy
		- Deliver static file (as css/js/image)
		- Load balancing

#### Remove the limitation in kernel stack
	- To be able to make good use of nginx. we need remove default configuration unnecessary(of course if you know how to nginx configuration :D)
	- /etc/sysctl.conf
		- net.core.somaxconn : 
		- net.ipv4.ip_local_port_range
		- sys.fs.file_max
		- net.ipv4.tcp_wmem & net.ipv4.tcp_rmem
		- Recommend:
			```nginx
				net.ipv4.ip_local_port_range='1024 65000'
				net.ipv4.tcp_tw_reuse='1'
				net.ipv4.tcp_fin_timeout='15'
				net.core.netdev_max_backlog='4096'
				net.core.rmem_max='16777216'
				net.core.somaxconn='4096'
				net.core.wmem_max='16777216'
				net.ipv4.tcp_max_syn_backlog='20480'
				net.ipv4.tcp_max_tw_buckets='400000'
				net.ipv4.tcp_no_metrics_save='1'
				net.ipv4.tcp_rmem='4096 87380 16777216'
				net.ipv4.tcp_syn_retries='2'
				net.ipv4.tcp_synack_retries='2'
				net.ipv4.tcp_wmem='4096 65536 16777216'
				vm.min_free_kbytes='65536'
			```


#### Log nginx to find out bottle neck
	- Have simple tool use be to do that:
	- [https://github.com/matsuu/kataribe](https://github.com/matsuu/kataribe)
	- you need setting nginx log format use directive
		```log
		log_format with_time '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" $request_time';
    	access_log /var/log/nginx/access.log with_time;
    	```

#### Caching with nginx
	- Nginx when use server static file, notice the setting about cache, compression. setting use gzip for static file is important
	- use gzip reduce cost relate IO, and though. Setting cache control will help server don't request static file loaded until cache expire.
		```conf
		http {
	    gzip              on;
	    gzip_http_version 1.0;
	    gzip_types        text/plain
	                      text/html
	                      text/xml
	                      text/css
	                      application/xml
	                      application/xhtml+xml
	                      application/rss+xml
	                      application/atom_xml
	                      application/javascript
	                      application/x-javascript
	                      application/x-httpd-php;
	    gzip_disable      "MSIE [1-6]\.";
	    gzip_disable      "Mozilla/4";
	    gzip_comp_level   1;
	    gzip_proxied      any;
	    gzip_vary         on;
	    gzip_buffers      4 8k;
	    gzip_min_length   1100;
	    ```

#### Advance nginx
	- use keepalive: keep alive is a technique of http to `keep` connection TCP even HTTP connection session is over, in order to reuse next request. This Technique very helpful when an use have a lot request to get static resource
	- ![keepalive](https://github.com/mui-le/blog/blob/master/nginx_advance.jpg)
	- add directive keepalive to upstream section
		```
		upstream app {
	      	server 127.0.0.1:5000;
	      	keepalive 16;
	    }

	


