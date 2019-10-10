# map的实现

Go的map底层使用hashmap实现的，由hash值mod当前hash表大小查找hash桶的位置。  
hash冲突是通过tophash的数组+overflow链表来存储的。BUCKETSIZE=8，也就是每个bucket存放8个key/value对，超出后申请新的bucket通过overflow形成链。  
bucket内部的查找是通过hash值的高8位存储在tophash中然后通过比较查找到位置，然后比较data中的key如果一样则找到对应的值，否则查找overflow bucket。  
这里面通过hash高位来比较可以加快比较，定位到key之后再进行key的比较。  
当hash冲突过多的时候通过扩容hashmap的大小来减低hash冲突提高查找性能。  
 ```go
struct Hmap
{
	uint8   B;	// 可以容纳2^B个项
	uint16  bucketsize;   // 每个桶的大小

	byte    *buckets;     // 2^B个Buckets的数组
	byte    *oldbuckets;  // 前一个buckets，只有当正在扩容时才不为空
};
```
```go
struct Bucket
{
	uint8  tophash[BUCKETSIZE]; // hash值的高8位....低位从bucket的array定位到bucket
	Bucket *overflow;           // 溢出桶链表，如果有
	byte   data[1];             // BUCKETSIZE keys followed by BUCKETSIZE values
};
```

hash key查找过程  
```go
do { //对每个桶b
	//依次比较桶内的每一项存放的tophash与所求的hash值高位是否相等
	for(i = 0, k = b->data, v = k + h->keysize * BUCKETSIZE; i < BUCKETSIZE; i++, k += h->keysize, v += h->valuesize) {
		if(b->tophash[i] == top) { 
			k2 = IK(h, k);
			t->key->alg->equal(&eq, t->key->size, key, k2);
			if(eq) { //相等的情况下再去做key比较...
				*keyp = k2;
				return IV(h, v);
			}
		}
	}
	b = b->overflow; //b设置为它的下一下溢出链
} while(b != nil);
```