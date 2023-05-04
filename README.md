Alexander Wilcox & Zhongwei Zhang
04/16/2023
CS 5700
Project #5

#_High-Level Approach_

We break down the program into three major components: implementation of HTTP server with outstanding caching 
algorithms, implementation of DNS server capable of responding to DNS queries, and dynamically mapping clients
to HTTP servers using an optimal content provider.

- In order to optimize HTTP servers response times, we managed to cache just under 20 MB of content on HTTP servers both in-memory and on the disk. This is achieved by creating serialized "pickle" files on the DNS disk during deployment, scp-ing that pickle file to HTTP servers, wiping that pickle file from the DNS servers, then downloading a 14 MB /cache/ on the DNS server to be scp'd during `runCDN`,
- DNS server is built utilizing the `dnslib` library and aims at responding to all Type A DNS queries with the correct domain name, 'cs5700cdn.example.com' in our case. If the query request contains unsupported domain names, the DNS server returns a root server with a 'NS' record. Multi-threading is used as a strategy to handle heavy parallel loads, with a maximum of two threads being allowed on the server (only two threads were used in consideration of RAM usage and operating performance),
- To find the optimal replica server, geographic location of the client is a major factor. With the help of the API service provided by Maxmind, we assign the closest server according to clients' ip addresses and corresponding geographic coordinates. However, to improve the stability of the DNS server and handle situations when the API service might down or be terminated due to unexpected reasons, we has a backup set of API credentials (utilized if an exception occurs when using the first set of credentials). In case both set of credentials fail, the `GeoInfo.py` class utilizes manually uploaded `csv` files containing geographic information as a last resort for the DNS service.

The program contains the following major components, including bash scripts, python source files, and `csv` data sheets:

`deployCDN`: A bash script used to upload relevant files from local to remote servers. As part of our caching 
	algorithm, it also helps create serialized "pickle" cache file on the DNS server and securely copy 
	that to the HTTP server, before deleting that serialized pickle file, using `scp` to transfer relevant 
	DNS files to the DNS server, and using the remaining 13 MB of DNS disk space to populate with a /cache/ 
	folder that will be `scp` to HTTP servers at `runCDN`.
`runCDN`: A bash script to start running the DNS server and the seven HTTP servers. This involves starting the 
	HTTP servers (which immediately read their pickle files into memory, delete those pickle files from their 
	disks, and start serving requests, all while downloading 7 MB worth of compressed articles into /cache/ in 
	another thread), and then starting the DNS server (which servers requests while using `scp` to transfer
	its 13 MB of /cache/ to the seven HTTP servers, before deleting its own copy of /cache/ entirely).
`stopCDN`: A bash script to terminate all running processes on the DNS server and HTTP servers and remove all 
	relevant files.
`httpserver`: A server which serves HTTP requests. `httpserver` utilizes both 18.5 MB of in-memory cache and 
	18.5 MB of disk cache via a `CacheManager` object to both deal with cached articles and handle cache 
	misses. `httpserver` handles GET requests by first evaluating if the requested path is "/grading/beacon", 
	then evaluating the validity of the path format, and then checking if the requested article is an in-memory 
	cache hit, else if it is an on-disk cache hit, else it fetches the article from the origin server via its 
	`CacheManager` object.
`CacheManager.py`: `CacheManager` is used to handle all caching related tasks. `CacheManager` is used by `httpserver` 
	for these purposes. `CacheManager` has static methods used to preemptively cache files during deployment on
	the disks of both the HTTP servers and the DNS server. `CacheManager` is also instantiated as an object by
	`httpserver` and handles article requests, as far as handling both cache hits and misses.
`build_in_memory_cache`: A simple executable program which writes a "pickle" file to disk; this pickle file contains
	the serialized contents of a Python dictionary which contains 18.5 MB of compressed article data. This pickle
	file is created during deployment, where it is created on the DNS server, transferred to the HTTP servers using
	`scp`, and then deleted from the DNS server.
`build_partial_disk_cache`: A simple executable program which writes 13.5 MB worth of compressed article data to the
	/cache/ folder on the DNS disk during deployment. Later, at runtime, this /cache/ folder is copied to the HTTP
	servers via `scp`, and then is deleted from the DNS server's disk. 
`utils.py`: A simple utilities files which contains functions used by `httpserver`.
`dnsserver`: The source file of the DNS server, which returns the IP address of the optimal replica server for clients. 
	Responses from the DNS server rely on the API service provided by Maxmind and public geoip databases as an 
	alternative approach.
`GeoInfo.py`: The file contains a class named 'GeoInfo', which contains geoip data by uploading `csv` files downloaded 
	from public databases before being treated for our purposes, and will be instantiated whenever the API service 
	by Maxmind is inaccessible.
`ip.csv`: A file which contains all ip data and corresponding country code (Source: IP2Location Database).
`coordinates.csv`: A file which contains data of country code and corresponding latitude and longitude information, and 
	which has been cleaned and treated in order to maintain an acceptable file size (Source: Google).

#_Who Worked On This Project_

- _Alexander_
  - Worked on the implementation of a HTTP server that fetches content from the origin on-demand in case of cache miss,
  - Built and optimized the caching mechanism on HTTP servers to optimize the cache hit ratio,
  - Worked on deploy/run/stopCDN scripts for CDN network operations.
- _Zhongwei_
  - Worked on the implementation of a system that maps IPs to nearby replica servers,
  - Worked on the implementation of a DNS server that dynamically returns IP addresses based on the mapping code,
  - Worked on runCDN scripts to initialize replica servers upon deployment.

#_Challenges_
There were many challenges that we ran into over the course of this project. Some include:

- Figuring out the way to `scp` files from one remote server to another remote server,
- Figuring out how to parse special characters in the paths of HTTP GET requests,
- Exploring how run `scp` operations in the background by using `stdout` redirection,
- Taking advantage of serialization to create a file that can be written into a Python dictionary and later read into one,
- Figuring out `pip install` in scripts for non-standard Python libraries,
- Handling limited HTTP disk space by fixing some bad paths for caching files,
- Solving the compress/decompress problem and exploring an efficient way to cache and read from the cache,
- Figuring out suitable API service for finding clients' geographic locations,
- Working on alternative approaches in the case of an API service crash by cleaning `csv` files and limiting `csv` files to an acceptable size,
- Figuring out how many threads should be allowed on the DNS server for maximum performance, and then optimizing DNS server source code in order to have more efficient RAM usage and quicker response rate,
- Figuring out the origin server is 'cs5700cdnorigin.ccs.neu.edu' instead of 'cs5700cdnorigin.css.neu.edu' - very HARDCORE challenge.

#_Testing_
For Project 5, testing is composed of in-progress testing and completion testing.

- In-Progress Testing:
  - HTTP Server: We tested against the server returned by `dig` by using `curl` and checking the accuracy of the returned HTML data,
  - DNS Server: We tested the DNS server by `dig` URLs with the 'cs5700cdn.example.com' domain; URLs without such domain are also tested and we checked if the DNS server returned the root server with an "NS" record,
  - Optimizing DNS Server: Several tests have been done in order to figure out how many threads should be allowed on the DNS server. A program named `measurement.py` (not included in the submission package) is constructed to send 100 requests in parallel to stress test the performance of DNS servers with different number of threads. Considering when max thread number is 2 the server has the best performance/lowest response time, we utilize the parameter of 2 in our final code,
- Completion Testing: We start running the DNS server and all HTTP servers and check the beacon status page. It shows they can operate as expected and showcase competitive performance compared to other groups.

#_Design Decisions_

- **DNS SERVER**
  - *DNS Server - Design Decisions*:
    During the development of the DNS server and GeoInfo class, several crucial design decisions were made. Firstly, we opted to 
    	utilize the MaxMind API to obtain the geographical coordinates of both clients and servers, ensuring accuracy and efficiency 
    	in implementation. Although a geo IP service Python library named geolite2 was considered for its ease of use, it was ultimately
    	discarded due to its inaccuracy in providing geographic information. Given that the API service enforces a 1,000 API request cap 
    	per day, we implemented two strategies to maintain the DNS server's sustainability and stability. Firstly, we cache server 
    	locations and client coordinates to conserve API usage. Secondly, we devised a fallback solution by implementing the GeoInfo class, 
    	which reads IP ranges and coordinates from CSV files, allowing the system to continue functioning even if the API becomes 
    	inaccessible. To achieve an acceptable file size for the CSV files, we removed unnecessary data columns and rounded numerical 
    	values to their nearest integers, effectively reducing the file size. As a result, the CSV files were downsized from approximately 
    	25 MB to only 6 MB. To boost performance, we employed `ThreadPoolExecutor` to handle DNS requests concurrently. Taking into account 
    	the limited RAM resources available and the potential overhead generated by additional threads, we restricted the maximum number 
    	of threads on the DNS server to two. This limitation has been proven to produce the best performance and lowest average response 
    	time through stress testing on the DNS server.
  - *DNS Server - Evaluating Effectiveness*:
    In order to thoroughly evaluate the effectiveness of our solution, we implemented a range of testing strategies. Firstly, to test the 
    	reliability of our fallback mechanism, we deliberately made the API inaccessible, simulating a scenario where the GeoInfo class would 
    	be required to function independently. This assessment allowed us to determine the robustness of our alternative approach in cases 
    	where the primary API might become unavailable. Secondly, we kept the DNS server operational and actively monitored the beacon status
    	page. This continuous monitoring allow us to see the comparison of our server's performance with that of other groups, providing valuable
    	insights into the efficiency and competitiveness of our implementation. Thirdly, we employed the dig command to submit DNS queries both 
    	containing correct domain names and those without. This step aimed to examine the server's ability to handle a diverse range of queries, 
    	ensuring that it could manage varying request types. Lastly, to exam the DNS server's capacity to withstand unexpected surges in parallel 
    	requests, we designed a custom stress testing program. This test simulated high-load situations, enabling us to assess the server's 
    	resilience and overall performance under challenging conditions. Although the DNS server displayed slightly better performance in the 
    	single-threaded scenario compared to the max-2-threads case, it is essential to consider that our testing was conducted using queries from 
    	only one IP address. This approach significantly simplifies the complexity of the testing environment. Therefore, we still recommend a
    	dding an additional thread to handle situations where the server might experience heavy loads of requests originating from various locations.
    	Test results as shown below:
|                       | No Multi-threading        | Max 2 Threads             | Max 8 Threads                      | Max 100 Threads                    |     |     |     |     |     |
| --------------------- | ------------------------- | ------------------------- | ---------------------------------- | ---------------------------------- | --- | --- | --- | --- | --- |
| Test 1 (1000 queries) | Missing Response Detected | Missing Response Detected | Missing Response Detected / 0.62MB | Missing Response Detected / 0.76MB |     |     |     |     |     |
| Test 2 (100)          | Avg 0.179531, 0.31MB      | Avg 0.278716, 0.37MB      | Avg 0.314031, 0.60MB               | Avg 0.439464, 0.78MB               |     |     |     |     |     |
| Test 3 (100)          | Avg 0.259303, 0.31MB      | Avg 0.198860, 0.36MB      | Avg0.407458, 0.61MB                | Avg 0.496252, 0.80MB               |     |     |     |     |     |
|                       |                           |                           |                                    |                                    |     |     |     |     |     |
  - *DNS Server - If Given More Time*:
    If given more time, there are two primary areas where improvements could be made in our processes. Firstly, when determining the optimal 
    	replica server to be assigned to a client, we would consider incorporating the Border Gateway Protocol (BGP) policy. By doing so, we would 
    	be able to factor in not only the geographic distance between the server and the client but also the time taken for data to travel across 
    	various networks. Currently, our system solely relies on the geographic proximity of the server and the client, which may not always result 
    	in the most efficient allocation. Secondly, with more time at our disposal, we would conduct a more comprehensive data cleaning process on 
    	our prepared `csv` files, aiming to reduce their file size even further. As it stands, the cumulative size of our `csv` files is 
    	approximately 6 MB. However, we believe that a more ideal and manageable size would be under 2 MB. A more efficient and thorough data 
    	cleaning process would enable us to save more on-disk space and ultimately enhance the overall performance and usability of our system.
  
- **HTTP SERVER**
  - *HTTP Server - Design Decisions*:
    There were a few important design decisions made while creating `httpserver`. First of all, we decided to relegate some utility functions to
    	`utils.py`. We also decided that `httpserver` should really only be concerned with serving GET requests; therefore, we decided to handle
    	cache hits and cache misses with `CacheManager` from `CacheManager.py`. In addition to `CacheManager` having some static methods used
    	outside of `httpserver`, the `CacheManager` object instance that `httpserver` uses contains a Python dictionary for in-memory cache 
    	(of size 18.5 MB), as well as knowledge of the on-disk cache (i.e., /cache/), as well as possessing the ability to fetch articles in
    	case of cache misses. Third, We used `ThreadingMixIn` to enable multi-threading on the `httpserver`. Finally, one more thing worth
    	mentioning is that We designed `httpserver` to immediately start serving requests, but also so that it deployed another `daemon` thread
    	before doing so, so that it could download 7 MB worth of compressed article files to its on-disk cache (i.e., /cache/) upon invocation.
  - *HTTP Server - Evaluating Effectiveness*:
    In order to ensure that `httpserver` efficiently served HTTP GET request, we employed a few methods. First of all, we did everything we could
    	to optimize the cache hit ratio. That required several tricks discussed elsewhere in this file. We also tried putting `httpserver` under 
    	duress by making it take full advantage of its multi-threading capability by sending it many requests at a time. We also studied different 
    	ways of filling its caches (both on-disk and in-memory) as fast as possible once `runCDN` was called, finally settling on our current 
    	implementation. We also used the /grading/beacon page as a cache hit agnostic view of our overall performance. 
  - *HTTP Server - If Given More Time*:
    If given more time, there were two main things we would have liked to implement on `httpserver`. First of all, at runtime all of the `httpserver` 
    	processes on each HTTP server launches another thread (apart from the main thread which serves GET requests) which downloads the remaining
    	7 MB of compressed article files in the local /cache/ folder (recall that the other 13 MB is being copied from the DNS server via `scp`).
    	This means that all 7 HTTP servers are all downloading the same 7 MB of contents from the origin server at the exact same time. Therefore,
    	we would have liked to have each HTTP server download ~1 MB of compressed article files into /cache/, where each of those files are disjoint
    	with the other ~1 MB of files being downloaded by the other 6 HTTP servers. Then, each HTTP server could `scp` its 1 MB of compressed article
    	files in its /cache/ to the other 6 HTTP servers. This would have put far less strain on the origin server. The second thing we would have
    	liked to do was create a custom `pageviews.csv` which re-computed the rankings of articles. In particular, we would have created this custom
    	`pageviews.csv` file ahead of time and then just used it going forward. To create this custom `pageviews.csv` file, we would have downloaded
    	every single compressed article file, and then for each article file, we would compute its total views divided by its compressed size; this
    	way, we could have cached the files that were the most economical. For example, suppose that we could only cache 2 MB of content; if the 
    	article ranked #1 got 999 views and had compressed size 2 MB, the article ranked #2 got 998 views and had compressed size 1 MB, and the 
    	article ranked #3 got 997 views and had compressed size 1 MB, then clearly you would get a higher cache hit rate by caching articles #2 and 
    	#3. However, our current program would cache only article #1, which is inefficient. Therefore, creating our custom `pageviews.csv` file would
    	have increased our cache hit ratio, thus leading to better performance. 
  
#_Demo of Program_
To be 100% sure that this program works at the very least, please find a demo of this program in the Google Drive folder linked below:
- https://drive.google.com/drive/folders/1currQcN61CKEzu3v-IXMOCmEEHYNWZ0a?usp=sharing

For reference, we created this Google Drive folder before submitting. The video you will find placed in this folder is a screen recording of the following steps:
1. Google today's date and time to prove that this video was created before 04/16/2023 11:59PM,
2. Download the Project 5 code that was submitted to Gradescope along with this `README.md` file),
3. Run `./deployCDN ...`,
	- `ssh` into `cdn-dns.5700.network` and do `ls -la` to demonstrate that all files were `scp` to the DNS server upon deployment, and that disk quota not exceeded with `du -sh *`,
	- `ssh` into `cdn-http1.5700.network` and do `ls -la` to demonstrate that all files were `scp` to the HTTP1 server upon deployment, and that disk quota not exceeded with `du -sh *`,
	- `ssh` into `cdn-http4.5700.network` and do `ls -la` to demonstrate that all files were `scp` to the HTTP4 server upon deployment, and that disk quota not exceeded with `du -sh *`,
	- `ssh` into `cdn-http6.5700.network` and do `ls -la` to demonstrate that all files were `scp` to the HTTP6 server upon deployment, and that disk quota not exceeded with `du -sh *`,
4. Run `./runCDN ...`,
	- Send a `dig` from my local machine to the DNS server,
	- Send several different `curl` requests to a few different HTTP servers,
	- `ssh` into `cdn-dns.5700.network`, do `ls -la` to demonstrate that all files were `scp` to the DNS server upon deployment, show that disk quota not exceeded with `du -sh *`, and do `ps aux | grep python`,
	- `ssh` into `cdn-http2.5700.network`, do `ls -la` to demonstrate that all files were `scp` to the HTTP2 server upon deployment, show that disk quota not exceeded with `du -sh *`, and do `ps aux | grep python`,
	- `ssh` into `cdn-http3.5700.network`, do `ls -la` to demonstrate that all files were `scp` to the HTTP3 server upon deployment, show that disk quota not exceeded with `du -sh *`, and do `ps aux | grep python`,
	- `ssh` into `cdn-http5.5700.network`, do `ls -la` to demonstrate that all files were `scp` to the HTTP5 server upon deployment, show that disk quota not exceeded with `du -sh *`, and do `ps aux | grep python`,
5. Run `./stopCDN ...`,
	- `ssh` into `cdn-dns.5700.network`, do `ls -la` to demonstrate that the folder is basically empty, and do `ps aux | grep python` to show that the server process has stopped,
	- `ssh` into `cdn-http1.5700.network`, do `ls -la` to demonstrate that the folder is basically empty, and do `ps aux | grep python` to show that the server process has stopped,
	- `ssh` into `cdn-http5.5700.network`, do `ls -la` to demonstrate that the folder is basically empty, and do `ps aux | grep python` to show that the server process has stopped,
	- `ssh` into `cdn-http7.5700.network`, do `ls -la` to demonstrate that the folder is basically empty, and do `ps aux | grep python` to show that the server process has stopped.


