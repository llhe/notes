Data Type Binary Representation
==============================

机器编码
--------------------------
主要目的是利于硬件实现  
  1. 整型  
  big/little-endian，补码形式  
  2. 浮点型  
  IEEE 745 standard  
  3. 字符串  
  实际上机器不理解字符串，属于语言层次的东西:   
    * c string
    * buf + len

SQLite4 ordered format
--------------------------
主要目的是索引有序，且不要求存储层理解数据类型  
  0. 数据类型编码  
  各种数据类型的第一位均为表示数据类型，保证了自解释性，支持peek，skip。
<table>
   <tr>
      <td>Content Type</td>
      <td>Encoding</td>
   </tr>
   <tr>
      <td>NULL</td>
      <td> 0x05</td>
   </tr>
   <tr>
      <td>NaN</td>
      <td> 0x06</td>
   </tr>
   <tr>
      <td>negative infinity</td>
      <td> 0x07</td>
   </tr>
   <tr>
      <td>negative large</td>
      <td> 0x08, ~E, ~M</td>
   </tr>
   <tr>
      <td>negative medium</td>
      <td> 0x13-E, ~M</td>
   </tr>
   <tr>
      <td>negative small</td>
      <td> 0x14, -E, ~M</td>
   </tr>
   <tr>
      <td>zero</td>
      <td> 0x15</td>
   </tr>
   <tr>
      <td>positive small</td>
      <td> 0x16, ~-E, M</td>
   </tr>
   <tr>
      <td>positive medium</td>
      <td> 0x17+E, M</td>
   </tr>
   <tr>
      <td>positive large</td>
      <td> 0x22, E, M</td>
   </tr>
   <tr>
      <td>positive infinity</td>
      <td> 0x23</td>
   </tr>
   <tr>
      <td>text</td>
      <td> 0x24, T</td>
   </tr>
   <tr>
      <td>binary</td>
      <td> 0x25, B</td>
   </tr>
   <tr>
      <td>final binary</td>
      <td> 0x26, X</td>
   </tr>
</table>
  另外，HBase-8201增加了固定长度整型和浮点型的编码  
  2. 排序方式支持  
  此处描述的编码方式是ASCENDING，DESCENDING方式对所有位取反  
  3. integer (fixed 32/64, 8/16同理, from orderly)  
  符号位取反即可  
  4. float, double (float32, float64)  
  5. numeric  
  6. binary var  
  第一位为类型，最后一位为0x00。当中位最高位总是1，即每个字节只有7位用来表示数据(空间开销是2B+1/8 ~ 12.5%)。可见编码解码过程是一个移位操作。  
  7. binary copy  
  如果需要编码的数据保证不包含0x00，则可以采用此种方式，以0x00表示结束，即跟c string一致，只是多了第一位数据类型字节  


proto buffer like encoding
----------------------------
此编码方案的主要目的是减少空间占用

