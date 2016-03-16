## 国内机票 PNR eterm 解析
* 解析 返回的 **RT** 信息 和 **PAT** 信息
* 解析 **不一定** 完全

### 以PNR为例:

  > **ELECTRONIC TICKET PNR**                                                     
1.曹泥美
2.曹尼玛 JVGG53                                                       
3.  HU7603 K   SU17JAN16PEKSHA RR2   2045 2255          E T1T2                 
4.SZV/T SZV/T0512-66666666/SUZHOU JK BUSINESS CONSULTING CO.,LTD/HUANG QIAN         ABCDEFG                                                                    
5.END                                                                          
6.T                                                                            
7.SSR FOID HU HK1 NI430923197705305423/P1                                      
8.SSR FOID HU HK1 NI342124197306051214/P2                                      
9.SSR ADTK 1E BY SZV17JAN16/1914 OR CXL HU7603 K17JAN                        
10.SSR TKNE HU HK1 PEKSHA 7603 K17JAN 8801664816611/1/P2                      
11.SSR TKNE HU HK1 PEKSHA 7603 K17JAN 8801664816612/1/P1                      
12.OSI HU CTCT18962129036                                                      
13.RMK TJ SZV173                                                               
14.RMK AO/ABE/KH-/OP-NN /DT-20160117190911                                     
15.RMK CA/NH66SX                                                               
16.RMK AUTOMATIC FARE QUOTE                                                    
17.FN/A/FCNY990.00/SCNY990.00/C0.00/XCNY50.00/TCNY50.00CN/TEXEMPTYQ/ACNY1040.00
18.TN/880-1664895527/P1                                                        
20.FP/CASH,CNY                                                                 
21.SZV173

### 预处理

       
```javascript

//1.将字符串中的空格符和回车符替换成空格符; 
	RT.replace(/[\n\r]+/g,' ');
//2.给字符串前端添加2个空白符(理由在解析乘客姓名的时候会说)
	RT += '  ' + RT;

```
### 1. 获取乘客姓名

```javascript

//1.通过pnr将RT信息拆分成2组,取拆分后的第一组,即为乘客信息;
	PSG = RT.split(this.pnr)[0];
//2.拆分乘客,通过  XX. 来拆分出各个乘客的姓名;(去除第一个是因为,一些RT信息前面有一些前缀,所以在预处理的时候增加了2个空格);
	passengers = RT.split(/[\s+-]\d+\./).slice(1)
	//我这里是通过 空格+-加一个数字加一个.来拆分的,
	  也可以在拆分成2组的时候将PSG中空格全部除去,然后通过一个数字一个.来拆分
//3.如果乘客是婴儿或者儿童,会在名字后面紧跟着INF和CHD
	可以自己通过拆分来判断；
	----- ？-----------
	在这里,我碰到了第一个问题;
	如果：姓名为 QIAN/CHD ,而他是一个儿童,那么解析出来的字符串应该是 QIAN/CHDCHD
	那么：QIAN/CHDCHD  他到底是一个完整的名字呢 还是是一个名字+儿童呢？
//4.解析乘客姓名完成;

```

### 2. 获取航程信息

```javascript

例子中下述行(仅第一行)描述了行程信息:(其他是可能出现的特殊情况)
    HU7603 K   SU17JAN  PEKSHA RR2   2045 2255          E T1T2   
    *HU7603 K   SU17JAN16PEKSHA RR2   2045 2255          E T1T2 (带*号为共享航班)
    HU7603 K   SU17JAN16PEKSHA RR2   2045 2255          E T1T2 (16,表示16年)
1.通过正则表达式匹配出所有满足的行,即所有行程;（上述是我目前碰到的情况,所以我只是针对他们写了一个正则,可能存在问题）
	var expg = /\s*([\w\*]{6,7})\s+(\w{1,2})\s+(\w{7})\d{0,2}\s*(\w{6})[\s*]*(\w{3})[\s*]*(\d{4})\s*(\d{4})[\+1]*\s*(E)\s*([-\w]{0,2})([-\w]{0,2})/g
	//使用/g模式匹配出所有满足行程格式的行/
2.在将匹配出的行分别拆分出各个信息;
	var exp = /\s*([\w\*]{6,7})\s+(\w{1,2})\s+(\w{7})\d{0,2}\s*(\w{6})[\s*]*(\w{3})[\s*]*(\d{4})\s*(\d{4})[\+1]*\s*(E)\s*([-\w]{0,2})([-\w]{0,2})/
3.解释下各个字段的意思:
	{
		*：共享航班
		HU  :航司二字码,
		7603:flightNo,
		K:cabin
		SU:week,
		17:day,
		JAN:month,
		PEK:depart,
		SHA:arrive,
		RR2:unKnow, //如果为HN1,NO1说明没座位了,客服妹子是这样说的~让我过滤并提醒一下;
		2045: 20:45 Time,
		2255: 22:55 Time,
		E: unKnow,
		T1T2: airportTower,  //如果没有则空,如果出发没有 则 --T2 ;
	}
4.解析航程信息完成;

```

### 3.获取证件信息

```javascript

例中:
	SSR FOID HU HK1 NI430923198805305523/P1
	
1.依旧是找到所有表示证件信息的所在行;
	var expg = /FOID([\s\w])+\/P\d/g ;
2.拆分出所有行中的证件号:
	var expg = /[A-Z]{2}(\w+)\/P(\d)/;
3.解释
{
	FOID :指令,
	HU:航司二字码,
	HK1:unknow,
	NI:unknow,
	430923198805305523:证件号,
	/P1:表示这是第一个乘客的证件,
}

```

### 4.获取价格信息

```javascript

例中: 
	例1: FN/A/FCNY990.00/SCNY990.00/C0.00/XCNY50.00/TCNY50.00CN/TEXEMPTYQ/ACNY1040.00
	例2: PAT:A 01 H FARE:CNY1370.00 TAX:CNY50.00 YQ:TEXEMPTYQ TOTAL:1420.00 SFC:01 SFN:01
说明:如果已出票 那么RT信息中会包含价格信息,如例1,出票中的价格在PAT信息中,如例2;

!!!注意:建议先将TEXEMPTYQ替换成0.00;下文不在提示;!!!

RT信息价格解析方法：
	1.依旧是解析出所有符合价格的行;
		由于我所做的功能的需求,暂时只会出现一条价格信息,所以我没有解析出行,只是单行,这条正则自行解决;
	2.解析出各价格的正则表达式;
	  //从RT信息中解析价格;
      var facePrice      = price.match(/FCNY(\d+.\d+)/);  //票面价
      var salePrice      = price.match(/ACNY(\d+.\d+)/);  //价格总和
      var airPortCstFee  = price.match(/(\d+.\d+)CN/);    //机场建设费
      var fuelFee        = price.match(/(\d+.\d+)YQ/);    //燃油附加费

PAT信息中解析方法:
	1.也是解析出所有的行;
	  这个一时兴起,我写了O(∩_∩)O哈哈~
	  var exp = /FARE((?!FARE).)*/g;
	2.解析出各价格的正则表达式;
	  从RT信息中解析价格;
	  var facePrice = price.match(/FARE:CNY(\d+.\d+)/);      //票面价
      var airPortCstFee = price.match(/TAX:CNY(\d+.\d+)/);   //价格总和
      var fuelFee = price.match(/YQ:(\d+.\d+)YQ/);           //机场建设费
      var salePrice = price.match(/TOTAL:(\d+.\d+)/);        //燃油附加费
	 
```

### 5.获取票号

```javascript

	由于我的功能不需要此字段,所以我并没有做解析;但是大致方法雷同;
	10.SSR TKNE HU HK1 PEKSHA 7603 K17JAN 8801664816611/1/P2                      
	11.SSR TKNE HU HK1 PEKSHA 7603 K17JAN 8801664816612/1/P1 
	例中上述2条代表着票号;

```

### 最后附上一张RT信息大致翻译(重要信息为虚拟,保护个人营私),以及我的javascript代码;

```javascript

 1.呵呵大 KXXV7C                                                                
 2.  FM9073 B   TH21JAN  SHAYNT HK1   0815 1005          E T2--                 
 3.SZV/T SZV/T0512-66666666/SUZHOU JK BUSINESS CONSULTING CO.,LTD/HUANG QIAN    
     ABCDEFG                                                                    
 4.END                                                                          
 5.TL/0715/21JAN/SZV173                                                         
 6.SSR FOID FM HK1 NI320520199907234221/P1                                      
 7.SSR CKIN FM                                                                  
 8.SSR FQTV FM HK1 SHAYNT 9073 B21JAN MU610285078316/P1                         
 9.SSR ADTK 1E BY SZV18JAN16/1616 OR CXL FM9073 B21JAN                          
10.OSI FM CTCT18962129036                                                       
11.RMK AO/ABE/KH-/OP-NN /DT-20160228791655                                      
12.RMK CA/PW5SZ9                                                                
13.SZV173   

1.	姓名  小编号
2.	航班号 仓位 星期日月 城市对 HK1起飞时间 降落时间 电子票标识 起飞航站楼—到达航站楼  
3.	订票地 订票公司电话号码/公司英文翻译/公司开台时留的人员信息（公司简介）
4.	自由格式（备注的信息，无意义）
5.	出票时限
6.	SSR FOID 航空公司代码 HK/NI证件号/旅客序号
7.	自由格式（备注的信息，无意义）
8	SSR FQTV 航空公司代码 HK1城市对 航班号 舱位起飞日期 航空公司代码 卡号/旅客序号 
9	SSR ADTK 1E BY 出票城市代码 客票保留截止日期/截止时间  or 取消 航班号 仓位定坐的日期
10	OSI 航空公司代码 CTCT电话号码
11	RMK  自由格式 （备注的信息，无意义）   
12	RMK CA/航司大编
13	订票公司的办公室号  

``` 

# Javascript

```javascript

function fltPnrTransFormService($q){

      this.get = get;

      function get(PNR,RT,PAT){
        var deferred = $q.defer();

        //解析数据
        var pnrClass       = new pnrClass(PNR,RT,PAT);   //创建PNR对象
        var tripsData      = pnrClass.getTrips()         //获得行程信息
        var passengersData = pnrClass.getPassengers()    //获得乘客信息
        var idcsData       = pnrClass.getIdcs()          //获得证件信息
        var priceData      = pnrClass.getPrice()         //获得价格信息

        //返回promise;
        return deferred.promise;

        //Pnr类;
        function pnrClass(pnr,rt,pat){
          //初始化
          this.pnr = pnr;
          this.rt = '   ' + rt.replace(/[\n\r]+/g,' '); //替换rt信息中的回车符;
          this.pat = pat;
          //数据解析

          //1.乘客解析
          this.getPassengers = function(){

            try{
              //开始解析

              //拆分总数据, 根据pnr拆分出乘客部分,乘客部分为 拆分后数组的第一个字符串;
              var passengers = '  ' + _.trim(this.rt.split(this.pnr)[0]) + ' ';

              //如果拆分的长度少于2说明PNR与RT信息不符;
              if(this.rt.split(this.pnr).length<2){
                alert('PNR与RT信息不相符合!');
                return null;
              };

              //根据  Num. 拆分出乘客姓名;
              passengers = _.compact(passengers.split(/[\s+-]\d+\./).slice(1));

              //清楚乘客名字符串中的空格;
              passengers = _.map(passengers,function(passenger){
                return _.trim(passenger)
              });

              //解析结束返回乘客名数组['XXXX','XXXX']
              return passengers;

            }catch(error){

              //乘客解析出错
              alert('乘客解析出错!');
              console.error('乘客解析出错:',error);

            };

          }; //end getPassengers

          //2.行程解析
          this.getTrips = function(){

            try{

              //用于判断此PNR是否为误操作造成的;
              var NOSITE = ['HN1','NO1'];

              //匹配出行程的正则表达式;
              var exp = /\s*([\w\*]{6,7})\s+(\w{1,2})\s+(\w{7})\d{0,2}\s*(\w{6})[\s*]*(\w{3})[\s*]*(\d{4})\s*(\d{4})[\+1]*\s*(E)\s*([-\w]{0,2})([-\w]{0,2})/;
              var exp_g = /\s*([\w\*]{6,7})\s+(\w{1,2})\s+(\w{7})\d{0,2}\s*(\w{6})[\s*]*(\w{3})[\s*]*(\d{4})\s*(\d{4})[\+1]*\s*(E)\s*([-\w]{0,2})([-\w]{0,2})/g;

              //匹配出所有行程 [行程1,行程2]
              var trips = this.rt.match(exp_g);
              if(!trips){
                alert('行程字符串捕获失败!');
              };
              //解析各行程;
              trips = _.map(trips,function(trip){

                //拆分各行程;
                var trip = trip.match(exp);

                //判断是否为误操作PNR;
                if(_.indexOf(NOSITE,trip[5])!==-1){
                  alert('此PNR显示无座位,错误位置:['+trip[0]+']');
                };

                //创建行程对象;
                var record = {
                  airlineCode :        trip[1].length===7?trip[1].slice(1,3):trip[1].slice(0,2),
                  flightNo:            trip[1].length===7?trip[1].slice(3):trip[1].slice(2),
                  cabin :              trip[2],
                  takeOffTime :        trip[3],
                  departAirportCode :  trip[4].slice(0,3),
                  arriveAirportCode :  trip[4].slice(3),
                  departTime :         trip[6],
                  arriveTime :         trip[7],
                  departAirportTower : trip[9],
                  arriveAirportTower : trip[10],
                };

                return record;

              });

              return trips;

            }catch(error){

              //行程解析出错
              alert('行程解析出错!');
              console.error('行程解析出错:',error);

            }
          }; //end getTrips

          //3.证件解析
          this.getIdcs = function(){

            //证件判断正则:
            var IDC_EXP = {
              '1': /(^\d{15}$)|(^\d{18}$)|(^\d{17}(\d|X|x)$)/,  //身份证
              '2': /^[a-zA-Z0-9]{5,17}$/ ,                      //护照
              '3': /^[HMhm]{1}([0-9]{10}|[0-9]{8})$/,           //港澳通行证
              '4': /(^[0-9]{8}$)|(^[0-9]{10}$)/,                //台湾通行证
            };

            try{

              //拆分出 证件信息字符串;
              var idcs = this.rt.match(/FOID([\s\w])+\/P\d/g);

              //详细拆分每个字符串;
              idcs = _.map(idcs,function(idc){

                //拆分出代表证件号和所对乘客的index; example: idc === "FOID CZ HK1 NI620502198302140083/P1"
                var idc = idc.match(/[A-Z]{2}(\w+)\/P(\d)/);

                //创建证件对象;
                var card = {
                  idcNo:    idc[1] || null,
                  index:    idc[2] || null,
                  idcType:  null,
                };

                //判断证件类型;
                _.each(IDC_EXP,function(exp,type){
                  if(!card.idcType && card.idcNo.match(exp)){
                    card.idcType = type;
                  };
                });

                return card;
              });

              return idcs;

            }catch(error){

              //证件解析出错
              alert('证件解析出错!');
              console.error('证件解析出错:',error);

            };

          }; //end getIdcs

          //4.价格解析
          this.getPrice = function(){

            try{
              //开始解析

              //将表示为0元的TEXEMPT字符串转换为0.00;
              var price = this.rt.replace('TEXEMPT','0.00');

              //从RT信息中解析价格;
              var facePrice      = price.match(/FCNY(\d+.\d+)/);  //票面价
              var salePrice      = price.match(/ACNY(\d+.\d+)/);  //价格总和
              var airPortCstFee  = price.match(/(\d+.\d+)CN/);    //机场建设费
              var fuelFee        = price.match(/(\d+.\d+)YQ/);    //燃油附加费

              //价格都为空,则从PAT信息中解析价格;
              if(!(facePrice && salePrice && airPortCstFee && fuelFee)){

                if(!this.pat){
                  alert('无价格获取点!请检查RT信息和PAT信息是否包含价格信息!');
                };

                //拆分价格数据 X组;
                var price = this.pat.replace('TEXEMPT','0.00').match(/FARE((?!FARE).)*/g);

                //当前不支持多程默认用第一种;
                var price = price[0];

                var facePrice = price.match(/FARE:CNY(\d+.\d+)/);      //票面价
                var airPortCstFee = price.match(/TAX:CNY(\d+.\d+)/);   //价格总和
                var fuelFee = price.match(/YQ:(\d+.\d+)YQ/);           //机场建设费
                var salePrice = price.match(/TOTAL:(\d+.\d+)/);        //燃油附加费

              };

              var priceInfo = {
                facePrice:       Number(facePrice? facePrice[1] : 0),             //票面价
                salePrice:       Number(salePrice? salePrice[1]||0 : 0),          //价格总和
                airportCstFee:   Number(airPortCstFee? airPortCstFee[1]||0 : 0),  //机场建设费
                fuelFee:         Number(fuelFee? fuelFee[1]||0 : 0),              //燃油附加费
                // tax: record.match(/(XCNY)(\d+.\d+)/)||0,
                // 实收价格: record.match(/(SCNY)(\d+.\d+)/)||0,
                // 代理费: record.match(/(C)(\d+.\d+)/)||0,
              };

              return [priceInfo];

              //从PAT信息中解析价格;

            }catch(error){

              //价格解析
              alert('价格解析出错!');
              console.error('价格解析出错:',error);

            };

          };

        };

      };// end get

    };

```
