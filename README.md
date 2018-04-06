# 模拟登录12306购票

#### 登录前获取验证码

```python
get_url = 'https://kyfw.12306.cn/passport/captcha/captcha-image?login_site=E&module=login&rand=sjrand&'+str(random.random())
with open('cap.png','wb') as f:
    f.write(self.session.get(get_url,headers=self.headers).content)
im = Image.open('cap.png')
im.show()
code = input('输入验证码用，分开：')
codeList = [[43, 43], [94, 43], [167, 42], [247, 40], [32, 111], [110, 115], [183, 114], [263, 110]]
code = re.findall('\d', code)
capcode = ''
for i in code:
    capcode += str(codeList[int(i)])
cap = re.findall('\d+', capcode)
capCode = ''
for i in cap:
    capCode += i + ','
return capCode[0:-1]
```

#### 检查验证码

```python
code = self.get_captcha()
check_url = 'https://kyfw.12306.cn/passport/captcha/captcha-check'
post_data = {
    'answer': code, #验证码对应的位置
    'login_site': 'E',
    'rand': 'sjrand'
}
check_res = self.session.post(check_url,data=post_data,headers=self.headers)
print(check_res.text)
if "验证码校验成功" in check_res.text:
    return True
else:
    self.check_captcha()#验证码不正确继续请求验证码
```

#### 查看登录所需参数

```python
login_url = 'https://kyfw.12306.cn/passport/web/login'
post_data = {
    'username':self.username,
    'password':self.password,
    'appid':'otn'
}
```

#### 查看登录结果

```python
if '登录成功' in login_res.text:
    url = 'https://kyfw.12306.cn/otn/login/userLogin'
    res = self.session.post(url,data={
        "_json_att":''
    },headers=self.headers)
    # 身份信息认证1
    url2 = 'https://kyfw.12306.cn/passport/web/auth/uamtk'
    tk_pat = '"newapptk":"(.+)"}'
    tk = re.compile(tk_pat).findall(self.session.post(url2,data={
        'appid':'otn'
    },headers=self.headers).text)[0]
    # 身份信息认证2
    tk_url = 'https://kyfw.12306.cn/otn/uamauthclient'
    tk_res = self.session.post(tk_url,data={
        'tk':tk
    },headers=self.headers)
    #认证成功后访问个人中心 获取用户名
    centerurl = "https://kyfw.12306.cn/otn/index/initMy12306"
    center_res = self.session.get(centerurl,headers={
        'user-agent':'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36'
    })
    user= re.findall('<span style="width:50px;">(.+)</span>',center_res.text)[0]
    if user:#用户名存在 即为成功
        print('登录成功 用户名：%s' % user)
```

#### 请求车次信息

```python
date = input('请输入时间格式为：2018-01-11：')
from_=input('请输入发车城市')
to_= input('请输入目的城市')
self.date = '2018-04-04'
from_code = stations[from_]
to_code = stations[to_]
self.form_ =  from_
self.form_code =  from_code
self.to_= to_
self.to_code= to_code
url = 'https://kyfw.12306.cn/otn/leftTicket/queryO?leftTicketDTO.train_date={}&leftTicketDTO.from_station={}&leftTicketDTO.to_station={}&purpose_codes=ADULT'

url =url.format(date,from_code,to_code)
res = self.session.get(url,headers=self.headers)
table = PrettyTable(["车次", "出发时间","到达时间","运行时间","商务",'一等','二等','软卧','动卧','硬卧','软座','硬座','无座'])
if '"flag":"1"' in res.text:
    result = res.json()
    for i in result['data']['result']:
        n = i.split('|')
        table.add_row([str(n[3]),str(n[8]),str(n[9]),str(n[10]),str(n[32]),str(n[31]),str(n[30]),str(n[23]),str(n[27]),str(n[28]),str(n[25]),str(n[29]),str(n[26])])
        # print('车次为：' + str(n[3]) + '  出发时间：  ' + str(n[8]) + ' 到达时间： ' + str(n[9]) + '运行时间：' + str(n[10]))
    print(table)
    num = input('输入车次:')
    for i in result['data']['result']:
        n = i.split('|')
        if n[3] == num.upper():
            secretStr =n[0]
            stationTrainCode = n[3]
            return secretStr,stationTrainCode
else:
    sleep(3)
    print('获取车票信息错误 重试。。。')
    self.get_tickets()
# 0= secretStr 3 = 车次 8=出发时间 9=到达时间 10 = 运行时间
#  32商务  31一等 30二等 23软卧 27动卧 28硬卧 25软座 29硬座 26无座  22其他
```

#### 购票请求

**准备信息**

```python
'''
设置购票的用户
:param name: 姓名
:param id: 身份证号码
:return:
'''
self.user=name
self.user_id=id
```

1. 检查用户信息

   ```python
   checkuser = 'https://kyfw.12306.cn/otn/login/checkUser'
   check_res = self.session.post(checkuser,data={
       '_json_att':''
   },headers=self.headers)
   ```

2. 提交订单请求

   ```python
   submit_url = 'https://kyfw.12306.cn/otn/leftTicket/submitOrderRequest'
   self.headers.update({
       'Referer': 'https://kyfw.12306.cn/otn/leftTicket/init',
   })
    submit_data={
                   'secretStr':urllib.parse.unquote(secretStr),
                   'train_date':self.date,
                   'back_train_date':self.date,
                   'tour_flag':'dc',
                   'purpose_codes':'ADULT',
                   'query_from_station_name':self.form_,
                   'query_to_station_name':self.to_,
                   'undefined':''
               }
               submit_res = self.session.post(submit_url,data=submit_data,headers=self.headers)
   ```

3. 用户信息检查

   ```python
   passenger_initdc_url = 'https://kyfw.12306.cn/otn/confirmPassenger/initDc'
   passenger_initdc_res = self.session.post(passenger_initdc_url,data={
       '_json_att':''
   },headers=self.headers).text
   #从这个页面获取 下面提交订单时 要带的参数
   token =re.findall('''globalRepeatSubmitToken = '(.+)';''',passenger_initdc_res)[0]
   train_no =re.findall("null,'train_no':'([0-9A-Za-z]+)','useMasterPool'",passenger_initdc_res)[0]
   leftTicket = re.findall("'leftTicketStr':'(.+)','limitBuy",passenger_initdc_res)[0]
   train_location = re.findall("'train_location':'(.+)'};",passenger_initdc_res)[0]
   key_check_isChange = re.findall("'key_check_isChange':'([0-9A-Za-z]+)','leftDetails':",passenger_initdc_res)[0]
   ```

4. 购票人信息检查

   ```python
   check_order_url='https://kyfw.12306.cn/otn/confirmPassenger/checkOrderInfo'
   check_order_data = {
       'cancel_flag':2,
       'bed_level_order_num':'000000000000000000000000000000',
       'passengerTicketStr':'3,0,1,'+self.user+',1,'+self.user_id+',,N',
       'oldPassengerStr':self.user+',1,'+self.user_id+',1_',
       'tour_flag':'dc',
       'ranCode':"",
       'whatsSelect':1,
       '_json_att':'',
       'REPEAT_SUBMIT_TOKEN':token,
   }
   check_order_res = self.session.post(check_order_url,data=check_order_data,headers=self.headers)
   ```

5. 价格信息获取

   ```python
   getQueueCount_url = 'https://kyfw.12306.cn/otn/confirmPassenger/getQueueCount'
   getQueueCount_data = {
       'train_date': 'Wed Apr 04 2018 00:00:00 GMT+0800 (中国标准时间)',
       'train_no': train_no,
       'stationTrainCode': stationTrainCode,
       'seatType': 3,
       'fromStationTelecode': self.form_code,
       'toStationTelecode': self.to_code,
       'leftTicket': leftTicket,
       'purpose_codes': '00',
       'train_location': train_location,
       '_json_att': '',
       'REPEAT_SUBMIT_TOKEN': token
   }
   self.headers.update({
       'Referer': 'https://kyfw.12306.cn/otn/confirmPassenger/initDc'
   })
   getQueueCount_res = self.session.post(getQueueCount_url,data=getQueueCount_data,headers=self.headers)
   ```

6. 提交订单

   ```python
   confirm_url = 'https://kyfw.12306.cn/otn/confirmPassenger/confirmSingleForQueue'
   confirm_data = {
       'passengerTicketStr':'3,0,1,'+self.user+',1,'+self.user_id+',,N',
       'oldPassengerStr': self.user+',1,'+self.user_id+',1_',
       'randCode':'',
       'purpose_codes':'00',
       'key_check_isChange':key_check_isChange,
       'leftTicketStr':leftTicket,
       'train_location':train_location,
       'choose_seats':'',
       'seatDetailType':'000',
       'whatsSelect':1,
       'roomType':'00',
       'dwAll':'N',
       '_json_att':'',
       'REPEAT_SUBMIT_TOKEN':token
   }
   confirm_res = self.session.post(confirm_url,data=confirm_data,headers=self.headers)
   # 如果返回 提交状态为 true  即为购票成功
   if '"submitStatus":true' in confirm_res.text:
       print('购票成功')
   ```

   # 这样就登录成功并且可以买票了
