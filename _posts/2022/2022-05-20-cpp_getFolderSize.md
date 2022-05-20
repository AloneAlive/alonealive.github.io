---
layout: single
title:  C++ 分区、文件夹大小获取、文件数据操作demo示例
date:   2022-05-20 10:19:02 +0800 
categories: cpp 
tags: cpp
toc: true
---

> Android C++模块有时候需要对文件系统进行操作，比如获取某个分区的大小、可用空间，获取某个路径文件夹的大小，文件内容读取及字符串比较、文件大小读取等demo代码示例

# 1. 获取分区大小和可用空间

```cpp
//方式3：使用statfs （头文件#include <sys/vfs.h>）类似df -h只能获取分区
#include <sys/vfs.h>
#include <stdio.h>

int main()  
{
	struct statfs diskInfo;
	statfs("/home/data",&diskInfo);
    
	unsigned long long blocksize = diskInfo.f_bsize;// 每个block里面包含的字节数
	unsigned long long totalsize = blocksize*diskInfo.f_blocks;//总的字节数
	unsigned long long availableDisk = diskInfo.f_bavail * blocksize; //可用空间

	char totalsize_GB[10]={0};
        printf("TOTAL_SIZE == %llu KB  %llu MB  %llu GB\n",totalsize>>10,totalsize>>20,totalsize>>30); // 分别换成KB,MB,GB为单位
	sprintf(totalsize_GB,"%.2f",(float)(totalsize>>20)/1024);
	printf("totalsize_GB=%s\n",totalsize_GB);

	return 0; 
}
```

***

# 2. 获取文件夹大小

代码如下：

```cpp
#include <stdio.h>  
#include <errno.h>  
#include <unistd.h>  
#include <sys/types.h>  
#include <sys/stat.h> 
#include <ftw.h>
#include <string>
#include <sys/vfs.h>

long long int totalDirectorySize; 

int sumDirectory(const char *fpath, const struct stat *sb, int typeflag) 
{ 
	totalDirectorySize += sb->st_size;  
	return 0; 
}  
 
long long int GetDirectorySize(const char* dir)
{
	totalDirectorySize = 0;
	
	if (!dir || access(dir, R_OK)) { 
		return -1; 
	} 
	
	if (ftw(dir, &sumDirectory, 1)) { 
		perror("ftw"); 
		return -2; 
	}
	
	return totalDirectorySize;
}
```

```cpp
//方式1：输入文件夹路径
/*int main(int argc, char **argv)  
{

	long long int total = GetDirectorySize(argv[1]);
 
	printf("%s: %lld\n", argv[1], total); 
	return 0; 
}*/

//方式2:直接指定某个文件夹
int main()  
{
    string logpath = "/home/user/Documents/test";

	long long int total = GetDirectorySize(logpath.c_str());
 
	printf("%s: %lld\n", logpath.c_str(), total); 
}
```

***

# 3. 删除路径文件

```cpp
//removeFileinPath.cpp 
#include <stdio.h>  
#include <errno.h>  
#include <unistd.h>  
#include <sys/types.h>  
#include <sys/stat.h> 
#include <string>
#include <sys/vfs.h>
#include <string.h>
#include <iostream>
#include <stdio.h>
#include <unistd.h>
#include <fstream>
#include <dirent.h>
#include <sys/statfs.h>
#include <errno.h>

using namespace std;

//remove all file in path
int main()  
{
    std::string logpath = "/home/user/Documents/demo/file/testlog";
    printf("%s\n", logpath.c_str()); 

    DIR *dir;
    struct dirent *ptr;

    if ((dir=opendir(logpath.c_str())) == NULL) {
        printf("logpath %s open failed %d error %s\n",logpath.c_str(),errno,strerror(errno));
        return 0;
    }

    //cndir(dir);  //set work dir

    while ((ptr=readdir(dir)) != NULL) {
        printf("-------------ptr->d_name=%s, ptr->d_type=%d\n", ptr->d_name, ptr->d_type);
        if((strcmp(ptr->d_name,".") == 0) || (strcmp(ptr->d_name,"..") == 0))
            continue;
        else if(ptr->d_type == 8) {
		//printf("remove file ! remove() or DeleteFile(/.log)\n");
                char removePath[256];
                sprintf(removePath, "%s/%s", logpath.c_str(), ptr->d_name);

                if (remove(removePath) != 0) {
                    printf("remove %s failed, %s\n", removePath,strerror(errno));
                } else {
                    printf("remove %s success\n", removePath);
                }
		}
    }
    closedir(dir);
	return 0; 
}
```

***

# 4. 文件行读取即字符串内容比较

**文件内容：**

```markdown
//testFileContext.txt
Version=1.0.1
MD5=12345678
```

**代码：**

```cpp
//readtestFileContext.cpp 
#include <stdio.h>
#include <malloc.h>
#include <vector>
#include <string>
#include <cstring>
#include <iostream>
#include <fstream>
#include <algorithm>

#define BYTES4_TO_U32(one,two,three,four)((((((unsigned int)one)<<24) | (unsigned int)two<<16)|(unsigned int)three<<8)|(unsigned int)four)

using namespace std;

int main() {
    std::string file_md5, file_version;
    std::string md5 = "12345678";
    char* md5_file_path;

    md5_file_path = "testFileContext.txt";

    printf("md5 file path------------------%s\n", md5_file_path);

    std::ifstream fp(md5_file_path, std::ios::in);

    if (!fp.is_open()) {
        printf("open failed\n");
        return 0;
    }

    getline(fp, file_version);
    getline(fp, file_md5);

    transform(file_md5.begin(), file_md5.end(), file_md5.begin(), ::tolower);

    printf("version of file ----------- %s\n", file_version.c_str());
    printf("md5 is------------------ %s\n", md5.c_str());
    printf("md5 of file------------------ %s\n", file_md5.c_str());

    if (file_md5.find(md5) != std::string::npos) {
        printf("md5 check success!\n");
        return 1;
    }
    printf("md5 check failed!\n");
    return 0;
}
```

**执行结果：**

```shell
$ ./readtestFileContext 
md5 file path------------------testFileContext.txt
version of file ----------- Version=1.0.1
md5 is------------------ 12345678
md5 of file------------------ md5=12345678
md5 check success!
```

***

# 5. 传输百分比计算

```cpp
//getPercent.cpp 
#include <stdio.h>
#include <string.h>

int main() {
	unsigned char percent = 0; 
	int dataNum = 3840;

	for (int index = 1; index <= dataNum; index++) {
		percent = index * 100 / dataNum;
    		if (index > 0 && percent == 0) {
    		    percent = 1;
		}
		printf("index = %d, percent = %d\n", index, percent);
	}

	return 0;
}
```

***

# 6. char字符数组打印

```cpp
printCharArr.cpp 
#include <stdio.h>
#include <string>

bool write_data(int fd, const void *buf, size_t nbyte) {
	printf("OK!");
}

int main() {
	static char status[11] =
                {0x5A, 0x05, 0x0B, 0x06, 0xB0, 0x01, 0x01, 0x00, 0x00, 0x00, 0x22};
	std::string s;
	int count = sizeof(status);
	for (int i=0; i < count;i++) {
		printf("%x  ", status[i]);
	}
	printf("\n");

	int fd=1;
	if (write_data(fd, status, sizeof(status))) {
		printf("call success\n");
	} else {
		printf("call error\n");
	}

	printf("status = %x %x\n", status[0], status[1]);
	printf("string = %s, count=%d\n", s.c_str(), count);
	return 0;
}
```

**执行结果：**

```shell
$ g++ printCharArr.cpp -o printCharArr
$ ./printCharArr 
5a  5  b  6  ffffffb0  1  1  0  0  0  22  
OK!call success
status = 5a 5
string = , count=11
```

***

# 7. 读取buffer字符串

```cpp
//readBufferString.cpp 
#include <stdio.h>
#include <string.h>
#include <malloc.h>
#include <vector>

#define BYTES4_TO_U32(one,two,three,four)((((((unsigned int)one)<<24) | (unsigned int)two<<16)|(unsigned int)three<<8)|(unsigned int)four)

using namespace std;

int main() {
	printf("Enter ...\n");
        char buffer[1024] = {'0', '/', 'd', 'a', 't', 'a', '/', 'b', 'u', 'f', 'f', 'e', 'r'};

        //memset(buffer, "0/data/buffer", sizeof(buffer));
        int len = 18;
        buffer[18] = '\0';
        int length = sizeof(buffer);
        printf("receive message: %s, %d, %d\n", buffer, len, length);

        return 0;
}
```

**执行结果：**

```shell
$ g++ readBufferString.cpp -o readBufferString
$ ./readBufferString 
Enter ...
receive message: 0/data/buffer, 18, 1024
```

***

# 8. bin二进制文件读取操作

```cpp
//readFileBin.cpp 
#include <stdio.h>
#include <string.h>
#include <malloc.h>
#include <vector>

#define BYTES4_TO_U32(one,two,three,four)((((((unsigned int)one)<<24) | (unsigned int)two<<16)|(unsigned int)three<<8)|(unsigned int)four)

using namespace std;

int main() {
	printf("test ascii");
	unsigned char send_buf[100];
	char buf[100];

        buf[0] = 1;
        send_buf[0] = buf[0] - 0x30;

	buf[1] = 3;
	send_buf[1] = buf[1] - 0x30;
	printf("RESLUT: send_buf[0]=%d, send_buf[1]=%d\n", send_buf[0], send_buf[1]);


	char ch1 = 8;
	char ch2 = 1;
	send_buf[2] = (ch1<<4) + ch2;
	printf("RESULT: send_buf[2] =%d\n", send_buf[2]);

	printf("Enter ...\n");
	FILE *fd=NULL;
	unsigned long fileLength = 0;
	long len = 1966080;	//0x0B 0x01 0x82 0x10 0x03 0x00 0x00 0x00 0x1E 0x00 0x00    //=1920kb偏移量之后的总大小
	long startAddress = 196608 ; //文件偏移量
	//1
	printf("0\n");
	fd = fopen("test.bin","r");
	printf("1\n");
	if (NULL == fd || !fd) {
		printf("can't open file\n");
		return -1;
	}
	printf("open file success\n");

	//2
	fileLength = len;   //file length 2112kb = 2162688 文件总大小
	//debug
	if(fseek(fd,0,SEEK_END) != 0) {
		printf("ReadFileToBuffer file seek end failed\n");
	}
	unsigned long debug_fileLength = ftell(fd);
	char *debug_buffer = NULL;
	debug_buffer = (char*)malloc(debug_fileLength+1);
	if(debug_buffer == NULL) {
		printf("malloc debug_buffer falied");
		return 0;
	}
	printf("debug buffer address is 0x%08x\n", debug_buffer);
	fseek(fd,0,SEEK_SET);
	int debug_readSize = fread(debug_buffer, 1, debug_fileLength, fd);
	if (ferror(fd)) {
		printf("read file failed\n");
	}

	printf("Debug Part: sum of file:%ld, startAddress:%ld, len:%ld, debug result debug_readSize:%d\n", debug_fileLength, startAddress, len, debug_readSize);
	printf("debug_buffer[0]=0x%02x, debug_buffer[1]=0x%02x\n", debug_buffer[0], debug_buffer[1]);


	//2162688-1966080=196608   --> 196608/1024 = 192
	//3
	//vector<unsigned char> tmp_buf[4] = {"0x10", "0x03", "0x00", "0x00"};  //file offeset
	//we need convert to 0x00 0x03 0x00 0x00
	unsigned int imageStartAddress = BYTES4_TO_U32(16, 3, 0, 0);
	imageStartAddress -= 0x10000000;
	printf("imageStartAddress=%d\n", imageStartAddress);
	startAddress = imageStartAddress;

	printf("3\n");
	if(fseek(fd, startAddress, SEEK_SET) != 0) {
   	    printf("fseek failed\n");
	}
	printf("fseek success\n");
	int n = ftell(fd);
	printf("n=%d\n", n);

	//4
	char *buffer;
	printf("4\n");
	buffer = (char*)malloc(fileLength+1);


	printf("malloc buffer\n");
	if (buffer) {
		printf("Memory alocation at %x\n", buffer);
	} else {
		printf("not enough memory");
		fclose(fd);
		free(buffer);
	}
	printf("malloc success\n");

	//5
	memset(buffer,0,fileLength);
	size_t readFileResult = fread(buffer, 1, fileLength, fd);
	printf("fileLength:%ld, readFileResult:%ld\n",fileLength, readFileResult);

	if(fileLength != readFileResult) {
		printf("fread file failed!!!!!! \n");
		fclose(fd);
		free(buffer);
		buffer = NULL;
		return -1;
	}
	printf("fread file success!\n");
	fclose(fd);
	return 0;
}
```

**执行结果：**

```shell
$ ./readFileBin 
test asciiRESLUT: send_buf[0]=209, send_buf[1]=211
RESULT: send_buf[2] =129
Enter ...
0
1
open file success
debug buffer address is 0x4a3db010
Debug Part: sum of file:2162688, startAddress:196608, len:1966080, debug result debug_readSize:2162688
debug_buffer[0]=0xffffff88, debug_buffer[1]=0x19
imageStartAddress=196608
3
fseek success
n=196608
4
malloc buffer
Memory alocation at 4a1fa010
malloc success
fileLength:1966080, readFileResult:1966080
fread file success!
```