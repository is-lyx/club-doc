---
sidebar: auto
---

# 社团管理系统 API 文档

本项目前后端接口规范和接口文档。

> 本文的正在持续更新中~

## 1 前后端接口规范

### 1.1 请求格式

采用 RESTful 风格的接口

#### 1.1.1 HTTP 动词（HTTP verbs）

|   Verb   |        Description         |
| :------: | :------------------------: |
|  `HEAD`  |     获取 HTTP 头部信息     |
|  `GET`   |          获取资源          |
|  `POST`  |          创建资源          |
| `PATCH`  | 更新部分 JSON 内容，不常用 |
|  `PUT`   |          更新资源          |
| `DELETE` |          删除资源          |

#### 1.1.2 分页（Pagination）

API 自动对请求的元素分页，不同的 API 有不同的默认值，可以指定查询的最大长度，但对某些资源不起作用。**请求后端数组数据时，统一传递四个分页参数**。

**Parameters**

| 参数    | 含义                                |
| ------- | ----------------------------------- |
| `page`  | 请求页码                            |
| `limit` | 每页的元素数量                      |
| `sort`  | 排序字段，例如 "add_time" 或者 "id" |
| `order` | 升序降序，只能是 "desc" 或者 "asc"  |

这里四个参数是可选的，后端应该设置默认参数，因此即使前端不设置， 后端也会自动返回合适的对象数组响应数据。

Example

```
GET /goods/list?page=1&limit=10
```

Response

```http
HTTP/1.1 200 OK

{
  "totalCount": 2525,
  "items": [
  ...
  ]
}
```

**返回参数说明**

响应体里面 `totalCount` 表示总数量，`items` 是数组，里面是要查询的元素。

### 1.2 响应格式

```json
Content-Type: application/json;charset=UTF-8

{
  body
}
```

响应成功，body 直接返回 JSON 格式的内容（返回列表时按 1.1.2 分页）：

```json
{
  "age": 12,
  "name": "zhangsan"
}
```

#### 1.2.1 失败异常

错误响应，`errors` 字段可选（一般是参数验证错误）

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "resource": "Issue",
      "field": "title",
      "code": "missing_field"
    }
  ]
}
```

**客户端错误（Client errors）**

接收请求 body 的 API 调用上可能存在三种类型的客户端错误：

1. 发送无效的 JSON 会导致 `400 Bad Request` 的响应

   ```http
   HTTP/1.1 400 Bad Request
   Content-Length: 35

   {"message":"Problems parsing JSON"}
   ```

2. 发送错误类型的 JSON 值将导致 `400 Bad Request` 的响应

   ```http
   HTTP/1.1 400 Bad Request
   Content-Length: 40

   {"message":"Body should be a JSON object"}
   ```

3. 发送无效字段将导致 `422 Unprocessable Entity` 的响应

   ```http
   HTTP/1.1 422 Unprocessable Entity
   Content-Length: 149

   {
     "message": "Validation Failed",
     "errors": [
       {
         "resource": "Issue",
         "field": "title",
         "code": "missing_field"
       }
     ]
   }
   ```

   所有错误对象都具有资源和字段属性，以便你的客户端可以知道问题所在。还有一个错误代码，让你知道该字段出了什么问题。下面这些是可能的验证错误代码：

   |    Error Name    |          Description           |
   | :--------------: | :----------------------------: |
   |    `missing`     |           资源不存在           |
   | `missing_field`  |    资源上的必填字段尚未设置    |
   |    `invalid`     |          字段格式无效          |
   | `already_exists` | 另一个资源与此字段具有相同的值 |

   资源也可能发送自定义验证错误（其中 `code` 是自定义的）。自定义错误将始终具有一个描述错误的 `message` 字段，并且大多数错误还将包括指向一些可能有助于您解决该错误的内容的 `documentation_url`字段。

**登录失败**

- 认证失败

```http
HTTP/1.1 401 Unauthorized

{
  "message": "Full authentication is required to access this resource"
}
```

- 没有权限

```http
HTTP/1.1 403 Forbidden

{
  "message": "Access is denied"
}
```

### 1.3 状态码（Status code）

| code | message               | description                                                                      |
| ---- | --------------------- | -------------------------------------------------------------------------------- |
| 200  | OK                    | 请求成功                                                                         |
| 201  | Created               | 创建了新资源，通常是 POST 请求                                                   |
| 400  | Bad Request           | 由于语法无效，服务器无法理解请求（错误的 JSON 格式）                             |
| 401  | Unauthorized          | 未认证错误                                                                       |
| 403  | Forbidden             | 客户端没有访问内容的权限                                                         |
| 404  | Not Found             | 找不到请求的资源；服务器也可以发送此响应而不是 403，以隐藏来自未授权客户端的资源 |
| 405  | Method not Allowed    | 一般是请求方法使用错误，eg. `POST` vs `GET`                                      |
| 429  | Too Many Requests     | 在给定的时间内发送了太多请求(速率限制)                                           |
| 500  | Internal Server Error | 服务器遇到了不知道如何处理的情况                                                 |
| 503  | Service Unavailable   | 服务器尚未准备好处理请求，常见原因是服务器因维护而停机或过载                     |

参考 [MDC web docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

### 1.4 认证方式（Authentication）

前后端采用 token 来验证访问权限。

#### 1.4.1 Header & Token

前后端 Token 交换流程如下：

1. 前端访问系统登录 API
2. 成功以后，前端会接收后端响应的一个 token，保存在本地
3. 请求受保护 API ，则采用自定义头部携带此 token
4. 后端检验 Token，成功则返回受保护的数据

#### 1.4.2 自定义 Header

访问受保护 API 采用自定义 `Authorization` 头部

1. 前端访问后端登录 API`/auth/login`

   ```json
   POST /user/login

   {
       "username": "admin",
       "password": "123456"
   }
   ```

2. 成功以后，前端会接收后端响应的一个 token

   ```json
   {
     "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0aGlzIGlzIGxpdGVtYWxsIHRva2VuIiwiYXVkIjoiTUlOSUFQUCIsImlzcyI6IkxJVEVNQUxMIiwiZXhwIjoxNTU3MzI2ODUwLCJ1c2VySWQiOjEsImlhdCI6MTU1NzMxOTY1MH0.XP0TuhupV_ttQsCr1KTaPZVlTbVzVOcnq_K0kXdbri0",
     "tokenHead": "Bearer"
   }
   ```

3. 请求受保护 API 则，则采用自定义头部携带此 token

   ```json
   GET http://localhost:8080/users
   Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0aGlzIGlzIGxpdGVtYWxsIHRva2VuIiwiYXVkIjoiTUlOSUFQUCIsImlzcyI6IkxJVEVNQUxMIiwiZXhwIjoxNTU3MzM2ODU0LCJ1c2VySWQiOjIsImlhdCI6MTU1NzMyOTY1NH0.JY1-cqOnmi-CVjFohZMqK2iAdAH4O6CKj0Cqd5tMF3M
   ```

#### 1.4.3 双重权限验证 :warning:

我们不仅要在接口上用注解控制访问权限，还可能要在业务逻辑中判断某个角色是不是对应改组织的角色。

举个例子：假如一个 A 社长在浏览器登录后，直接在浏览器地址栏输入删除 B 社团活动的链接，后端收到请求后，角色验证通过，但是 A 社长实际上不能操作其它社团的内容！

### 1.5 跨域资源共享（Cross origin resource sharing）

API 支持来自任何来源的 AJAX 请求的跨来源资源共享（CORS）

参考 [https://www.w3.org/TR/cors/](https://www.w3.org/TR/cors/)

## 2 用户服务 :hear_no_evil:

### 2.1 登录

用户登录接口

```
POST /auth/login
```

#### Input

| Name       | Type     | Description |
| :--------- | :------- | ----------- |
| `username` | `string` | 用户名      |
| `password` | `string` | 密码        |

#### Response

```json
{
  "token": "your secret token"
}
```

**注意：**登陆失败返回 400，token 过期返回 401

### 2.2 拉取用户信息

使用 token 拉取用户详细信息

> token 放在请求头

```
GET /auth/info
```

#### Response

```json
{
  "userid": 10088,
  "avatar": "https://wpimg.wallstcn.com/f778738c-e4f8-4870-b634-56703b4acafe.gif",
  "username": "test",
  "nickname": "Bob",
  "major": "software",
  "email": "bob@gmail.com",
  "slogan": "good good study, day day up",
  "phone": "12345678901",
  "roles": ["student"]
}
```

**注意**： roles 字段只返回 `student` / `admin`

### 2.3 注册

用户注册接口

```
POST /auth/register
```

#### Input

| Name       | Type     | Description |
| ---------- | -------- | ----------- |
| `username` | `string` | 用户名      |
| `password` | `string` | 密码        |
| `emaill`   | `string` | 邮箱        |
| `authCode` | `string` | 邮箱验证码  |

#### Response

```json
Status: 201 Created
```

### 2.4 通过密保问题找回密码

根据用户 ID 获取密保问题

```
GET /users/question
```

#### Parameter

| Name       | Type     | Description      |
| ---------- | -------- | ---------------- |
| `username` | `string` | **必填**；用户名 |

#### Response

```json
Status: 200 OK

{
  "loginProblem": "what is your name?"
}
```

### 2.5 校验密保问题

校验密保问题回答是否正确

```
POST /users/anwser
```

#### Input

| Name       | Type     | Description    |
| ---------- | -------- | -------------- |
| `username` | `string` | 用户名         |
| `anwser`   | `string` | 待校验密保答案 |

#### Response if answer is correct

```json
Status: 204 No Content
```

#### Response if answer is wrong

```json
Status: 403 Forbidden
```

### 2.6 修改个人信息

用户修改个人信息

```
PUT /users/info
```

#### Input

| Name        | Type     | Description |
| ----------- | -------- | ----------- |
| `nickname`  | `string` | 昵称        |
| `major`     | `string` | 专业        |
| `phone`     | `string` | 联系方式    |
| `address`   | `string` | 地址        |
| `slogan`    | `string` | 个人标语    |
| `email`     | `string` | 邮箱        |
| `avatarUrl` | `string` | 头像链接    |

#### Example

```json
{
  "nickname": "jack",
  "major": "software",
  "phone": "12345678901",
  "address": "fujian",
  "avatarUrl": "xxxx.png"
}
```

#### Response

```json
Status: 204 No Content
```



### 2.8 退出登陆

用户退出登陆

```
POST /auth/logout
```

### 2.9 修改用户密码

通过验证旧用户密码修改密码

```
POST /auth/password
```

#### Input

| Name          | Type     | Description |
| ------------- | -------- | ----------- |
| `username`    | `string` | 用户名      |
| `oldPassword` | `string` | 旧密码      |
| `newPassword` | `string` | 新密码      |

#### Response

```json
Status: 204 No Content
```

### 2.10 修改用户密码

忘记密码时通过回答保密问题修改密码

```
PUT /users/password
```

#### Input

| Name       | Type     | Description  |
| ---------- | -------- | ------------ |
| `username` | `string` | 用户名       |
| `password` | `string` | 新密码       |
| `answer`   | `string` | 密保问题答案 |

#### Response

```json
Status: 204 No Content
```



## 2.11 获取邮箱验证码

```
GET /auth/authCode
```

### Parameter

| Name    | Type     | Description    |
| ------- | -------- | -------------- |
| `email` | `string` | **必填**. 邮箱 |

#### Response

```
Status: 200 OK
{
	"message":  "获取验证码成功"
}
```



## 3 社团管理 :two_men_holding_hands:

### 3.1 社团推荐列表

获取系统推荐的社团列表

```
GET /clubs/recommended
```

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "name": "篮球社",
    "chiefName": "微微笑",
    "type": "运动",
    "state":"1",
    "avatarUrl": "xxxx.png",
    "joinState": "已加入",
    "role": "社长"
  }
]
```

### 3.2 社团列表

列出所有社团

```
GET /clubs?keyword=篮球
```

#### **Parameters**

| 参数    | 含义                                          |
| ------- | --------------------------------------------- |
| keyword | 可选，关键词，用名称模糊查找                  |
| type    | 可选，类型，用社团类型模糊查找                |
| state   | 可选，状态（0/1），用社团（官方）状态精确查找 |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "name": "篮球社",
    "chiefName": "微微笑",
    "type": "运动",
    "state":"1",
    "avatarUrl": "http://xx/xxxx.png",
    "joinState": "未加入",
    "role": "无"
  }
]
```

### 3.3 某个学生的社团列表（社员/社长）

列出学生加入的所有社团

```
GET /clubs/users/:id/clubs?status=member
```

#### Parameter

| 参数名 | 含义                                                   |
| ------ | ------------------------------------------------------ |
| status | 学生在社团身份，可取值为 member（成员），chief（社长） |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "name": "篮球社",
    "chiefName": "微微笑",
    "type": "运动",
    "state":"1",
    "avatarUrl": "http://xx/xxxx.png",
    "joinState": "已加入",
    "role": "社长"
  }
]
```

### 3.4 查看某个社团详情

根据社团 ID 查找社团

```
GET /clubs/:id
```

#### Response

```json
Status: 200 OK

{
  "id": 2,
  "name": "篮球社",
  "chiefName": "微微笑",
  "type": "运动",
  "state": "1",
  "avatarUrl": "http://xx/xxxx.png",
  "joinState": "已加入",
  "role": "社长",
  "slogan": "XX社是一个非常非常厉害的社团",
  "memberCount": 500,
  "qqGroup": "312512512"
}
```

### 3.5 查看学生加入社团申请列表

查看某个学生的加入社团申请列表，state 申请状态：0 -> 未审核; 1 -> 审核通过; 2 -> 审核未通过;

```
GET /clubs/users/joins
```

#### Response

```json
Status: 200 OK

[
  {
    "clubName": "篮球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "这是申请原因申请原因申请原因",
    "state": 0
  },
  {
    "clubName": "篮球社",
    "createAt": "2018-04-19 18:13:12",
    "reason": "这是申请原因申请原因申请原因",
    "state": 1
  }
]
```

### 3.6 查看学生创建社团申请列表

查看某个学生的创建社团申请列表，state 申请状态：0 -> 未审核; 1 -> 审核通过; 2 -> 审核未通过;

```
GET /clubs/users/creations
```

#### Parameters

组合查询参数

| 参数     | 含义                                               |
| -------- | -------------------------------------------------- |
| name     | 可选，关键词，用 `username` 或 `nickname` 模糊查找 |
| honor_id | 可选，头衔 ID                                      |

#### Response

```json
Status: 200 OK

[
  {
    "clubName": "羽毛球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "交朋友",
    "state": 0
  },
  {
    "clubName": "篮球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "交朋友",
    "state": 1
  }
]
```

### 3.7 社团成员列表

根据社团 ID 列出社团成员

```
GET /clubs/:clubId/members
```

### **Parameters**

| 参数    | 含义                           |
| ------- | ------------------------------ |
| name    | 可选，模糊查找昵称或用户名     |
| honorId | 可选，通过头衔 ID 精确查找头衔 |

#### Response

```json
Status: 200 OK

[
  {
    "userId": "20012",
    "username": "221701300",
    "nickname": "张三",
    "honor": "龙王",
    "role": "社长",
    "credit": "100",
    "avatarUrl": "https://xxx.com/images/xxxx.png"
  }
]
```

#### Response if requester is not an club member(TODO)

```
Status: 302 Found
```

### 3.8 社团成员详细信息

查看某个社团成员信息

```
GET /clubs/:clubId/members/:userId
```

#### Response

```json
Status: 200 OK

{
  "username": "xxx",
  "nickname": "张三",
  "slogan": "我只是一个测试的",
  "role": "社员",
  "major": "数计学院软件工程",
  "phone": "1231254125",
  "email": "1195669260@qq.com",
  "address": "@string",
  "honor": "龙王",
  "credit": 100,
  "avatarUrl": "https://xxx.com/images/xxxx.png"
}
```

#### Response if requester is not an club member(TODO)

```
Status: 302 Found
```

### 3.9 添加社团成员

添加社团成员需要学生[手动提交申请](#4.7 学生提交入社申请)，由社长审核后自动加入社团。

### 3.10 删除社团成员

为了删除用户在社团中的成员身份，经过身份验证的用户必须是社团所有者（社长）。

```
DELETE /clubs/:clubId/members/:userId
```

#### Response

```
Status: 204 No Content
```

### 3.11 修改社团信息

社长修改社团信息

```
PUT /clubs/:clubId/info
```

#### Input

均为选填，但至少要有一个非空参数

| Name      | Type     | Description |
| --------- | -------- | ----------- |
| `slogan`  | `string` | 社团简介    |
| `qqGroup` | `string` | QQ 群号     |
| `type`    | `string` | 社团类型    |

#### Example

```json
{
  "slogan": "test",
  "qqGroup": "112233",
  "type": "运动"
}
```

#### Response

```json
Status: 204 No Content
```

### 3.12 修改社团头像

社长修改社团头像

```
PUT /clubs/:clubId/pic
```

#### Input

| Name        | Type     | Description |
| ----------- | -------- | ----------- |
| `avatarUrl` | `string` | 社团头像    |

#### Example

```json
{
  "avatarUrl": "xxx"
}
```

#### Response

```json
Status: 204 No Content
```



### 3.13 修改社团头像（本地上传）

社长修改社团头像

```
POST /clubs/:clubId/info/avatar
```

#### Parameter

| Name    | Type            | Description |
| ------- | --------------- | ----------- |
| `image` | `multipartFile` | 图片        |

#### Example

```json
{
  "image": "头像文件"
}
```

#### Response

```json
Status: 200

{
  "avatarUrl": "新的的头像链接"
}
```

### 3.14 查看社团走马灯

查看某个社团走马灯图片

```
GET /clubs/:clubId/pictures
```

#### Response

```json
Status: 200 OK

[
  "链接1",
  "链接2",
  "链接3",
  "链接4",
  "链接5"
]
```


### 3.15 修改社团走马灯（本地上传）

社长修改社团走马灯图片

```
POST /clubs/:clubId/pictures
```

#### Parameter

| Name    | Type            | Description |
| ------- | --------------- | ----------- |
| `image` | `multipartFile[]` | 图片        |

#### Example

```json
{
  "image": [null,"data1",null,"data2","data3"]
}
```

#### Response

```json
Status: 200

{
  "url": [null,"data1",null,"data2","data3"]
}
```

## 4 申请与审核 :page_with_curl:

这里先统一定义一下申请状态（state）：

注意：**后端只返回申请状态码，前端需要自行映射！**

| 申请状态码 | 描述                 |
| ---------- | -------------------- |
| 0          | pending，未审核      |
| 1          | active，审核通过     |
| 2          | rejected，审核被拒绝 |

### 4.1 提交创建社团申请表单

普通学生提交社团创建申请表单

```
POST /clubs/creations
```

#### Input

| Name            | Type      | Description                                  |
| --------------- | --------- | -------------------------------------------- |
| `clubName`      | `string`  | 社团名称                                     |
| `reason`        | `string`  | 申请原因                                     |
| `type`          | `string`  | 社团类型                                     |
| `officialState` | `integer` | 社团官方状态，0 -> 小团体，1 -> 官方认证社团 |

#### Example

```json
{
  "clubName": "test",
  "reason": "make friends",
  "type": "运动类",
  "officialState": 1
}
```

#### Response

```json
Status: 201 Created
```

### 4.2 社团创建申请列表

管理员可查看社团创建申请列表，以进行进一步的审核

```
GET /clubs/creations
```

#### Parameters

以下参数用于组合查询
（原则上 4.1-4.17 接口中的查看列表功能，返回什么字段就可以使用什么字段进行多条件查询,以下不再列出可用于查询的参数列表）

| 参数名          | 参数类型 | 含义                                                   |
| --------------- | -------- | ------------------------------------------------------ |
| `id`            | Integer  | 申请 id                                                |
| `applicant`     | String   | 申请人                                                 |
| `clubName`      | String   | 社团名称                                               |
| `reason`        | String   | 申请理由                                               |
| `officialState` | Integer  | 官方状态: 0 -> 非正式; 1 -> 正式;                      |
| `createAt`      | String   | 申请时间                                               |
| `state`         | Integer  | 申请状态：0 -> 未审核; 1 -> 审核通过; 2 -> 审核未通过; |

#### Response

```json
Status: 200 OK
[
  {
    "id": 1,
    "applicant": "张三",
    "clubName": "羽毛球社",
    "reason": "交朋友",
    "officialState": 1,
    "createAt": "2018-04-19 18:14:12",
    "state": 0
  },
  {
    "id": 2,
    "applicant": "李四",
    "clubName": "乒乓球社社",
    "reason": "交朋友",
    "officialState": 0,
    "createAt": "2018-04-19 18:14:12",
    "state": 1
  },
  {
    "id": 3,
    "applicant": "王五",
    "clubName": "篮球社",
    "reason": "交朋友",
    "officialState": 0,
    "createAt": "2018-04-19 18:14:12",
    "state": 2
  }
]
```

### 4.3 审核创建社团申请

管理员审核某个社团创建申请

```
PUT /clubs/creations/audit
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Example

```json
{
  "id": 1,
  "state": 1
}
```

#### Response

```json
Status: 204 No Content
```

### 4.3 学生撤销创建社团申请

学生撤销某个社团创建申请

```
PUT /clubs/creations/revocation
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Example

```json
{
  "id": 1,
  "state": 1
}
```

#### Response

```json
Status: 204 No Content
```

### 4.4 提交解散社团申请

社长提交解散社团申请表单

```
POST /clubs/dissolution
```

#### Input

| Name           | Type      | Description |
| -------------- | --------- | ----------- |
| `clubId`       | `integer` | 社团 ID     |
| `applicant`    | `string`  | 申请人      |
| `reason`       | `string`  | 申请原因    |
| `accessoryUrl` | `string`  | 附件链接    |

#### Example

```json
{
  "id": 1,
  "clubId": 10001,
  "applicant": "张三",
  "reason": "没为什么",
  "accessoryUrl": "https://xxx/xxx/xx.doc"
}
```

#### Response

```json
Status: 201 Created
```

### 4.5 社团解散申请列表

管理员可查看社团解散申请列表，以进行进一步的审核

```
GET /clubs/dissolution
```

#### Parameters

返回参数都可用于组合查询

#### Response

```json
Status: 200 OK
[
  {
    "id": 1,
    "clubName": "羽毛球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "本社团没有存在意义",
    "state": 0
  },
  {
    "id": 2,
    "clubName": "足球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "好无聊啊",
    "state": 1
  },
  {
    "id": 3,
    "clubName": "羽毛球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "无",
    "state": 2
  }
]
```

### 4.6 审核社团解散申请

管理员审核某个社团解散申请

```
PUT /clubs/dissolution/audit
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Example

```json
{
  "id": 1,
  "state": 1
}
```

#### Response

```json
Status: 204 No Content
```

### 4.7 学生提交入社申请

学生提交入社申请表单

```
POST /clubs/join
```

#### Input

| Name     | Type      | Description |
| -------- | --------- | ----------- |
| `clubId` | `integer` | 社团 ID     |
| `reason` | `string`  | 申请原因    |

#### Example

```json
{
  "clubId": 5000,
  "reason": "没为什么"
}
```

#### Response

```json
Status: 201 Created
```

### 4.8 加入社团申请列表

社长可查看入社申请列表，以进行进一步的审核

```
GET /clubs/:id/joins
```

#### Parameters

返回参数都可用于组合查询

#### Response

```json
Status: 200 OK

[
    {
      "id": 1,
      "applicant": "wangs",
      "reason": "锻炼自己",
      "createAt": "2018-04-19 18:14:12",
      "state": 0
    },
    {
      "id": 2,
      "applicant": "li",
      "reason": "锻炼自己",
      "createAt": "2018-04-19 18:14:12",
      "state": 1
    },
    {
      "id": 3,
      "applicant": "zhao",
      "reason": "锻炼自己",
      "createAt": "2018-04-19 18:14:12",
      "state": 2
    }
]
```

### 4.9 审核入社申请

社长审核某个学生入社申请

```
PUT /clubs/joins/audit
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Response

```json
Status: 204 No Content
```

### 4.10 社员提交退社表单

社员提交退出社团表单

```
POST /clubs/quit
```

#### Input

| Name     | Type      | Description |
| -------- | --------- | ----------- |
| `clubId` | `integer` | 社团 ID     |
| `reason` | `string`  | 申请原因    |

#### Response

```json
Status: 201 Created
```

### 4.11 退社通知列表

社长可查看成员退出社团通知列表

```
GET /clubs/:id/quit
```

#### Parameters

返回参数都可用于组合查询

#### Response

```json
Status: 200 OK

[
  {
    "username": "wangs",
    "reason": "want to go",
    "createAt": "2018-04-19 18:14:12"
  },
  {
    "username": "li",
    "reason": "want to go",
    "createAt": "2018-04-19 18:14:12"
  }
]
```

### 4.12 社长提交换届申请

社长提交换届申请表单

```
POST /clubs/leader/change
```

#### Input

| Name           | Type      | Description |
| -------------- | --------- | ----------- |
| `clubId`       | `integer` | 社团 ID     |
| `newChiefName` | `string`  | 新社长姓名  |
| `reason`       | `string`  | 申请原因    |

#### Response

```json
Status: 201 Created
```

### 4.13 社长换届申请列表

管理员查看社长换届申请列表

```
GET /clubs/leader/changes
```

#### Parameters

返回参数都可用于组合查询

#### Response

```json
Status: 200 OK

[
  {
    "id":1,
    "clubName": "wangs",
    "oldChiefName": "want",
    "newChiefName": "zhang",
    "createAt": "2018-04-19 18:14:12",
    "state": 0
  },
  {
    "id":2,
    "clubName": "wangs",
    "oldChiefName": "want",
    "newChiefName": "zhang",
    "createAt": "2018-04-19 18:14:12",
    "state": 2
  }
]
```

### 4.14 换届申请审核

管理员可审核某个换届申请

```
PUT /clubs/leader/changes
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Response

```json
Status: 204 No Content
```

### 4.15 社团认证申请

社长提交社团认证申请表单

```
POST /clubs/certifications
```

#### Input

| Name     | Type      | Description |
| -------- | --------- | ----------- |
| `clubId` | `integer` | 社团 ID     |
| `reason` | `string`  | 申请原因    |

#### Example

```json
{
  "clubId": 1,
  "reason": "XXX"
}
```

#### Response

```json
Status: 201 Created
```

### 4.16 社团认证申请列表

管理员查看社长换届申请列表

```
GET /clubs/certifications
```

#### Parameters

返回参数都可用于组合查询

#### Response

```json
Status: 200 OK

[
  {
    "id":1,
    "clubName": "羽毛球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "xxx",
    "state": 0
  },
  {
    "id":2,
    "clubName": "羽毛球社",
    "createAt": "2018-04-19 18:14:12",
    "reason": "yyy",
    "state": 2
  }
]
```

### 4.17 社团认证申请审核

管理员可查看审核某个换届申请

```
PUT /clubs/certifications
```

#### Input

| Name    | Type      | Description  |
| ------- | --------- | ------------ |
| `id`    | `integer` | 申请 ID      |
| `state` | `integer` | 新的申请状态 |

#### Response

```json
Status: 204 No Content
```

### 4.18 本社团认证申请列表

社长查看本社团认证申请状态

```
GET /clubs/:clubId/certifications
```

#### Parameters

该接口不可进行组合查询

#### Response

```json
Status: 200 OK

total_count: 50
items: [
  {
    "id": 1,
    "reason": "书法社",
    "createAt": "2018-04-19 18:14:12",
    "state": 0
  }
]
```

### 4.19 本社换届申请列表

社长查看本社团换届申请状态

```
GET /clubs/:clubId/leaderChange
```

#### Parameters

该接口不可进行组合查询

#### Response

```json
Status: 200 OK

data: {
    "id": 1,
    "reason": "书法社",
    "createAt": "2018-04-19 18:14:12",
    "state": 0
  }
```

### 4.20 本社团解散申请列表

社长查看本社团解散申请状态

```
GET /clubs/:clubId/dissolution
```

#### Parameters

该接口不可进行组合查询

#### Response

```json
Status: 200 OK

data: {
    "id": 1,
    "reason": "书法社",
    "createAt": "2018-04-19 18:14:12",
    "state": 0
  }
```

## 5 公告

### 5.1 发布公告

社长发布一条公告

```
POST /clubs/:club/bulletins
```

#### Input

| Name     | Type     | Description |
| -------- | -------- | ----------- |
| `title`  | `string` | 公告标题    |
| `body`   | `string` | 公告内容    |
| `imgUrl` | `string` | 公告图片    |

#### Example

```json
{
  "title": "this is a bulletin",
  "body": "3 days later"
}
```

#### Response

```json
Status: 201 Created
```

### 5.2 查看公告列表

```
GET /clubs/:club/bulletins
```

#### Parameters

以下参数用于组合查询

| 参数名  | 参数类型 | 含义     |
| ------- | -------- | -------- |
| `title` | String   | 公告标题 |
| `body`  | String   | 公告内容 |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "title": "公告1",
    "body": "这是内容",
    "createAt": "2018-04-19 18:14:12",
    "updateAt": "2018-04-19 19:14:12"
  },
  {
    "id": 2,
    "title": "公告2",
    "body": "这是内容",
    "createAt": "2018-04-19 18:14:12",
    "updateAt": "2018-04-19 19:14:12"
  }
]
```

### 5.3 查看公告详情

```
GET /clubs/:clubId/bulletins/:id
```

#### Response

```json
Status: 200 OK

{
  "title": "公告1",
  "body": "这是内容",
  "createAt": "2018-04-19 18:14:12",
  "updateAt": "2018-04-19 19:14:12"
}
```

### 5.4 修改公告内容

```
PUT /clubs/:clubId/bulletins/:id
```

#### Input

| Name    | Type     | Description |
| ------- | -------- | ----------- |
| `title` | `string` | 公告标题    |
| `body`  | `string` | 公告内容    |

#### Example

```json
{
  "body": "更新后的内容"
}
```

#### Response

```json
Status: 204 No Content
```

### 5.5 删除某一条公告

根据公告 ID 删除某一条公告

```
DELETE /clubs/bulletins/:id
```

#### Response

```json
Status: 204 No Content
```

## 6 活动

活动状态: 0 -> “未审核”; 1 -> "审核通过"; 2 -> "已发布"; 3 -> "审核未通过"; 4 -> "已结束"; 5 -> "已删除"

### 6.1 申请活动

```
POST /clubs/activities
```

#### Input

| Name          | Type       | Description |
| ------------- | ---------- | ----------- |
| `clubId`      | `int`      | 社团编号    |
| `name`        | `string`   | 活动名称    |
| `title`       | `string`   | 活动标题    |
| `content`     | `string`   | 活动内容    |
| `startDate`   | `datetime` | 开始时间    |
| `endDate`     | `datetime` | 结束时间    |
| `location`    | `string`   | 地点        |
| `memberCount` | `int`      | 参与人数    |
| `imgUrl`      | `multipartFile`   | 图片        |

#### Example

```json
{
  "clubId": 1,
  "name": "act",
  "title": "this is a title",
  "content": "what content",
  "startDate": "2018-04-19 20:20:20",
  "endDate": "2018-04-22 20:20:10",
  "location": "三区",
  "memberCount": 22,
  "imgUrl": "https://xxx/xxx/a.png"
}
```

#### Response

```json
Status: 201 Created
```

### 6.2 管理员获取社团活动申请

```
GET /clubs/activities
```

#### Parameters

以下参数用于组合查询

| 参数名     | 参数类型 | 含义                                                                            |
| ---------- | -------- | ------------------------------------------------------------------------------- |
| `clubName` | String   | 社团名称                                                                        |
| `state`    | Integer  | 申请状态：0 -> 未审核; 1 -> 审核通过; 2 -> 已发布; 3 -> 审核未通过; 4 -> 已结束 |

#### Response

```json
Status: 200 OK

[
    "totalCount": 3,
    "items": [
        {
            "id": 3,
            "clubName": "足球社",
            "name": "act3",
            "title": "welcome to act3",
            "content": "this is amazing!",
            "startDate": "2020-04-25 23:47:21",
            "endDate": "2020-04-30 23:47:25",
            "location": "风雨操场",
            "state": 1
        },
        {
            "id": 4,
            "clubName": "足球社",
            "name": "act4",
            "title": "welcome to act4",
            "content": "this is amazing!",
            "startDate": "2020-04-25 23:47:21",
            "endDate": "2020-04-30 23:47:25",
            "location": "风雨操场",
            "state": 1
        },
        {
            "id": 5,
            "clubName": "软件学社",
            "name": "act5",
            "title": "welcome to act5",
            "content": "this is amazing!",
            "startDate": "2020-04-25 23:47:21",
            "endDate": "2020-04-30 23:47:25",
            "location": "风雨操场",
            "state": 1
        }
    ]
]
```

### 6.3 活动申请审核

管理员审核某个社团活动申请

```
PUT /clubs/activities/audit
```

#### Input

| Name    | Type      | Description       |
| ------- | --------- | ----------------- |
| `id`    | `integer` | 申请 ID，不可修改 |
| `state` | `integer` | 活动状态          |

#### Response

```json
Status: 204 No Content
```

### 6.4 修改社团活动状态

社长修改社团活动

```
PUT /clubs/activities/state
```

#### Input

| Name    | Type      | Description       |
| ------- | --------- | ----------------- |
| `id`    | `integer` | 申请 ID，不可修改 |
| `state` | `integer` | 活动状态          |

#### Response

```json
Status: 204 No Content
```

### 6.5 修改社团活动

社长修改社团活动

```
PUT /clubs/activities/:id
```

#### Input

| Name       | Type       | Description |
| ---------- | ---------- | ----------- |
| `name`     | `string`   | 活动名称    |
| `title`    | `string`   | 活动标题    |
| `content`  | `string`   | 活动内容    |
| `starDate` | `datetime` | 开始时间    |
| `endDate`  | `datetime` | 结束时间    |
| `location` | `string`   | 地点        |

#### Response

```json
Status: 204 No Content
```

### 6.6 删除某一活动

根据活动 ID 删除某一条活动，后端需要获取请求发送方的用户 id，判断他是否拥有待删除活动的权限

```
DELETE /clubs/activities/:id
```

#### Response

```json
Status: 204 No Content
```

### 6.7 获取活动申请列表

社长可以获取自己社团申请的活动列表

```
GET /clubs/:id/activities/apply
```

#### Parameters

以下参数用于组合查询

| 参数名      | 参数类型 | 含义                         |
| ----------- | -------- | ---------------------------- |
| `name`      | String   | 活动名称                     |
| `state`     | String   | 活动状态                     |
| `content`   | String   | 活动内容                     |
| `location`  | String   | 活动地点                     |
| `startDate` | String   | 活动开始时间 例如 2020-05-01 |
| `endDate`   | String   | 活动结束时间 例如 2020-05-01 |

#### Response

```json
Status: 200 OK

[
    "totalCount": 3,
    "items": [
        {
            "id": 10,
            "name": "act",
            "location": "三区",
            "content": null,
            "memberCount": null,
            "startDate": "2018-04-19 00:00:00",
            "endDate": "2018-04-22 00:00:00",
            "state": 0
        },
        {
            "id": 7,
            "name": "act7",
            "location": "风雨操场",
            "content": "this is amazing!",
            "memberCount": 22,
            "startDate": "2020-04-25 23:47:21",
            "endDate": "2020-04-30 23:47:25",
            "state": 2
        },
        {
            "id": 6,
            "name": "act6",
            "location": "风雨操场",
            "content": "this is amazing!",
            "memberCount": 22,
            "startDate": "2020-04-25 23:47:21",
            "endDate": "2020-04-30 23:47:25",
            "state": 1
        }
    ]
]
```

### 6.8 获取某活动申请详情

社长可以获取自己社团申请的某一活动的详情

```
GET /clubs/activities/apply/:id
```

#### Response

```json
Status: 200 OK

{
  "id": 1,
  "clubId": 5000,
  "name": "act1",
  "title": "welcome to act1",
  "body": "this is amazing!",
  "imgUrl": "https://unsplash.com/photos/tvc5imO5pXk",
  "starDate": "2020-04-25 23:47:21",
  "endDate": "2020-04-30 23:47:25",
  "location": "风雨操场",
  "memberCount": 22,
  "createAt": "2020-04-25 23:47:46",
  "handleAt": null,
  "state": 5
}
```

### 6.9 获取某活动帖推荐列表

| 参数名      | 参数类型 | 含义                         |
| ----------- | -------- | ---------------------------- |
| `number`      | Integer   | 获取推荐数量             |

注：默认获取十条，图片地址可能为空

```
GET /clubs/activities/recommended
```
#### Response

```json
Status: 200 OK
[
    {
        "id": 56,
        "imgUrl": ""
    },
    {
        "id": 4,
        "imgUrl": "https://images.unsplash.com/photo-1556035511-3168381ea4d4?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=967&q=80"
    },
    {
        "id": 55,
        "imgUrl": ""
    }
]
```

## 7 活动论坛

### 7.1 帖子列表

```
GET /forum/posts
```

#### Parameters

除了必填参数，其它均为组合查询参数

| 参数名       | 参数类型 | 含义                                               |
| ------------ | -------- | -------------------------------------------------- |
| `type`       | Integer  | **必填**；帖子来源：0 -> 个人帖；1 -> 社团活动帖； |
| `posterName` | string   | 发帖人                                             |
| `title`      | string   | 标题                                               |
| `content`    | string   | 内容                                               |
| `createAt`   | string   | 发帖日期，形如 `2018-04-19`                        |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "posterName": "发帖人",
    "avatafrUrl": "e312312312312.jpg",
    "title": "这是一个帖子",
    "content": "这是内容",
    "imgUrl": "131231241241.jpg",
    "createAt": "2018-04-19 18:14:12"
  }
]
```

### 7.2 查看某一帖子

```
GET /forum/posts/:id
```

#### Parameters

以下参数区分个人贴和社团活动帖

| 参数名 | 参数类型 | 含义                                               |
| ------ | -------- | -------------------------------------------------- |
| `type` | Integer  | **必填**；帖子来源：0 -> 个人帖；1 -> 社团活动帖； |

#### Response

```json
Status: 200 OK

{
  "id": 1,
  "posterName": "发帖人",
  "avatafrUrl": "e312312312312.jpg",
  "title": "这是一个帖子",
  "content": "这是内容",
  "imgUrl": "131231241241.jpg",
  "createAt": "2018-04-19 18:14:12",
  "commentCount": "3",
  "likeCount": 4
}
```

### 7.3 删除个人帖

只能删除自己发的帖子（逻辑删除）

```
DELETE /forum/posts/:id
```

#### Response

```json
Status: 204 No Content
```

### 7.4 发布个人帖

```
POST /forum/posts
```

#### Parameter

| 参数名    | 参数类型      | 含义     |
| --------- | ------------- | -------- |
| `title`   | string        | 帖子标题 |
| `content` | string        | 帖子内容 |
| `image`   | MultipartFile | 图片     |

#### Response

```
Status: 201 Created
```

### 7.5 修改个人帖

```
PUT /forum/posts
```

#### Input

```json
{
  "title": "这是一个帖子",
  "content": "这是内容"
}
```

#### Response

```
Status: 204 No Content
```

### 7.6 获取某一社团的帖子列表

```
GET /forum/:clubId/posts
```

#### Parameters

除了必填参数，其它均为组合查询参数

| 参数名       | 参数类型 | 含义                                               |
| ------------ | -------- | -------------------------------------------------- |
| `type`       | Integer  | **必填**；帖子来源：0 -> 个人帖；1 -> 社团活动帖； |
| `posterName` | string   | 发帖人                                             |
| `title`      | string   | 标题                                               |
| `content`    | string   | 内容                                               |
| `createAt`   | string   | 发帖日期，形如 `2018-04-19`                        |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "posterName": "发帖人",
    "avatafrUrl": "e312312312312.jpg",
    "title": "这是一个帖子",
    "content": "这是内容",
    "imgUrl": "131231241241.jpg",
    "createAt": "2018-04-19 18:14:12",
    "likeCount" 3
  },
  {
    "id": 2,
    "posterName": "发帖人",
    "avatafrUrl": "e312312312312.jpg",
    "title": "这是一个帖子",
    "content": "这是内容",
    "imgUrl": "131231241241.jpg",
    "createAt": "2018-04-19 18:14:12",
    "likeCount": 5
  }
]
```

### 7.7 个人帖子列表

查看我的帖子

```
GET /forum/posts/mine
```

#### Parameters

组合查询参数

| 参数名     | 参数类型 | 含义                        |
| ---------- | -------- | --------------------------- |
| `title`    | string   | 标题                        |
| `content`  | string   | 内容                        |
| `createAt` | string   | 发帖日期，形如 `2018-04-19` |

#### Response

```json
Status: 200 OK

[
  {
    "id": 1,
    "posterName": "发帖人",
    "avatafrUrl": "e312312312312.jpg",
    "title": "这是一个帖子",
    "content": "这是内容",
    "imgUrl": "131231241241.jpg",
    "createAt": "2018-04-19 18:14:12",
    "likeCount": 4
  }
]
```



## 7.8 点赞

对帖子进行点赞

```
POST /forum/like
```

### Parameters

| 参数名        | 参数类型 | 含义    |
| ------------- | -------- | ------- |
| `likedPostId` | BigInt   | 帖子 id |

### Response

```
Status: 204 No Content
```

注：若用户重复点赞，返回 `400` 错误



## 7.9 取消点赞

对帖子进行点赞

```
POST /forum/unlike
```

### Parameters

| 参数名        | 参数类型 | 含义    |
| ------------- | -------- | ------- |
| `likedPostId` | BigInt   | 帖子 id |

### Response

```
Status: 204 No Content
```

注：若用户未点赞，返回 `400` 错误



## 7.10 查看当前用户对某一帖子点赞情况

查看当前用户对某一帖子的点赞情况

```
GET /forum/like
```

### Parameters

| 参数名   | 参数类型 | 含义    |
| -------- | -------- | ------- |
| `postId` | BigInt   | 帖子 id |

### Response

```
Status: 200
{
	"status": 1
}
```

注：**0 -> 未点赞，1 -> 已点赞**



## 8 帖子评论

### 8.1 发表评论

对某一帖子发表评论

```
POST /forum/posts/remarks
```

#### Input

| Name      | Type      | Description |
| --------- | --------- | ----------- |
| `postId`  | `integer` | 帖子 ID     |
| `content` | `string`  | 评论内容    |

#### Example

```json
{
  "postId": 33,
  "content": "very good"
}
```

#### Response

```json
Status: 201 Created
```

### 8.2 某帖子的评论列表

```
GET /forum/posts/:id/remarks
```

#### Response

```json
Status: 200 OK

[
  {
    "useId": 1,
    "nickname": "wangp",
    "avatarUrl": "e312312312312.jpg",
    "content": "这是内容1",
    "createAt": "2018-04-19 18:14:12"
  },
  {
    "useId": 1,
    "nickname": "zhangs",
    "avatarUrl": "e312312312312.jpg",
    "content": "这是内容2",
    "createAt": "2018-04-19 18:14:12"
  }
]
```



## 8.3 删除某一评论

评论发布者可以删除自己的评论

```
DELETE /forum/posts/remarks/:id
```

#### Response

```
Status: 204 No Content
```



### 更新用户活跃度（后台自动）

### 更新用户头衔（后台自动）

## 9 管理员首页

### 9.1 待审核事项

```
GET /admin/unaudited
```

#### Response

```json
Status: 200 OK

[
  {
    "createNumber":1,
    "dismissNumber":2,
    "activityNumber":3,
    "changeNumber":4,
    "identifyNumber":5
  }
]
```

### 9.2 注册人数

```
GET /admin/newusers
```
### Parameters

| 参数名        | 参数类型 | 含义    |
| ------------- | -------- | ------- |
| `startDate` | string   | 开始日期 |
| `endDate` | string   | 结束日期 |

#### Response

```json
Status: 200 OK

[
  {
    "date":[2020-5-1,2020-5-2,2020-5-3,2020-5-4,2020-5-5,2020-5-6,2020-5-7,2020-5-8,2020-5-9,2020-5-10],
    "newUsers":[0,0,100,0,0,31,0,0,73,101]
  }
]
```
### 9.3 各类别社团数

```
GET /admin/clubspecies
```

#### Response

```json
Status: 200 OK

[
  {
    "clubSpecies":["艺术","运动","学习","休闲","其他"],
    "clubSpeciesNumber":[10,20,5,3,23]
  }
]
```

## 10 积分

## 积分规则

### 个人积分规则
每日签到：8积分

社团活动评论：5积分（每个活动仅限一次评论获取积分机会）

每日积分获取上限：50积分

每月清零

| 活跃度等级 | 描述                  |
| ---------- | -------------------- |
| 1          | 0 - 99积分           |
| 2          | 100 - 249积分        |
| 3          | 250 - 399积分        |
| 4          | 400 - 699积分        |
| 5          | 600 - 849积分        |
| 6          | 850以上积分          |

### 社团积分规则
发布活动帖：20积分

社团每个成员每日获取的积分将以1:1比例加入社团积分

|      等级 | 描述                  |
| ---------- | -------------------- |
| 1          | 0 - 999积分           |
| 2          | 1000 - 1999积分        |
| 3          | 2000 - 2999积分        |
| 4          | 3000 - 3999积分        |
| 5          | 4000以上积分          |



### 10.1 签到获取积分

```
POST /credit/:clubId/checkin
```

#### Response

```json
Status: 201 Created
```
### 10.2 查看签到状态

| 签到状态码 | 描述                  |
| ---------- | -------------------- |
| 0          | done，今日已经签到    |
| 1          | granted，今日可以签到 |
| 2          | denied，不可签到      |

```
GET /credit/:clubId/ischeckin
```

#### Response

```json
Status: 200 OK
[
  {
    "state": 1,
    "message":"可以签到"
  }
]
```

### 10.3 获取用户在某社团的积分和头衔

```
GET /credit/:clubId/userhonor
```

#### Response

```json
Status: 200 OK

  {
    "grade":"冒泡",
    "score": 6,
    "percentage": 30
  }

```


### 10.4 获取某社团的积分和头衔

```
GET /credit/:clubId/clubhonor
```

#### Response

```json
Status: 200 OK

  {
    "grade": 1,
    "score": 6,
    "percentage": 30
  }

```


### 10.5 获取积分和头衔规则（用户）

```
GET /credit/userhonor
```

#### Response

```json
Status: 200 OK
[
  {
    "grade":"潜水",
    "lowerlimit": 0,
    "upperlimit": 20
  },
  {
    "grade":"冒泡",
    "lowerlimit": 21,
    "upperlimit": 40
  }
]

```

### 10.6 获取积分和头衔规则（社团）

```
GET /credit/clubhonor
```

#### Response

```json
Status: 200 OK
[
  {
    "grade": 1,
    "lowerlimit": 0,
    "upperlimit": 20
  },
  {
    "grade": 2,
    "lowerlimit": 21,
    "upperlimit": 40
  }
]

```

