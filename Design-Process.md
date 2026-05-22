Phase1
  需求整理：
    - 用户分为：admin/user
    - user可以完成：登录/登出/注册/浏览商品/下单单个商品/支付/查看订单
    - admin可以完成：删除商品/更新商品/查看所有订单/新增商品
  
Phase2
  DB Design
    - User Table
      - id: PK
      - password_hash: varchar
      - user_name: varchar
      - email: varchar, unique, not null #当把其设为unique的时候，db会自动为其创建索引
      - created_at: timestamp
      - updated_at: timestamp
      - is_admin: bool
    
    - Product Table
      - id: PK
      - name: varchar
      - price: decimal(10, 2)
      - short_description: varchar
      - display_img_url: varchar
      - stock: int
      - status: active/inactive not null #需要status方便管理商品，admin可以快速上架下架某产品
      - created_at: timestamp
      - updated_at: timestamp
    
    - Order Table
      - id: pk
      - user_id: fk -> users.id
      - total_amount: decimal(10, 2)
      - created_at: timestamp
      - updated_at: timestamp
      - status: pending/paid/shipped/completed not null
    
    - OrderItem Table
      - id: pk
      - order_id: fk -> orders.id
      - product_id: fk
      - quantity: int
      - price: decimal(10, 2)
    
    - Payment Table
      - id: pk
      #- user_id: fk #可以不要，因为已经有order_id，可以从order table中获取user id
      - order_id: fk -> orders.id
      - payment_method: alipay/wechat/card (not null)
      - amount: decimal(10, 2)
      - status: pending/success/failed not null
      - created_at: timestamp
      - updated_at: timestamp

Phase3 
  API Design （Base Path /api/v0）
  
    - Auth API
      - /api/v0/auth/register [POST]
        request: email, password, user_name
        response: id, user_name, email, is_admin, created_at
      
      - /api/v0/auth/login [POST]
        request: email, password
        response: access_token, token_type??, expires_in?? --> access_token: JWT字符串（核心凭证）；token_type: 通常固定写 "bearer"（告诉客户端怎么放到 header）;expires_in：token 多少秒后过期（比如 1800 秒）
      
      - /api/v0/auth/logout [POST]
    
    - Product API
      - /api/v0/products [GET]
        query: page(默认1), page_size(默认10, 最大100) ??? --> 该route后面可加的查询参数 eg：/api/v0/products?page=1&page_size=10
        request:
        response: items[], total, page, page_size
      
      - /api/v0/products/{product_id} [GET]
        response: id, name, short_description, price, display_img_url, stock, status, created_at, updated_at
      
      - /api/v0/products/{product_id} [PUT] (admin) ???不需要传入userId，查询是否为admin吗 --> 不需要，jwt可解析当前用户身份
        request: 需更新的字段
        response：更新后的商品完整信息
      
      - /api/v0/products/{product_id} [DELETE] (admin)
        response: message
      
      - /api/v0/products [POST] (admin)
        request: 需要添加的商品信息
        response：message
    
    - Order API
      - /api/v0/admin/orders [GET] (admin：看全部订单)
        response：orders[], total, page, page_size
      
      - /api/v0/orders [GET] (user：只看自己的订单)
        response：orders[], total, page, page_size
      
      - /api/v0/orders/{order_id} [GET] (admin:可看所有， user:只看自己的)
        response：order完整信息
      
      - /api/v0/orders [POST] (user)
        request: product_id, quantity
        response：order_id, total_amount, order_status, items[], created_at
    
    - Payment API
      - /api/v0/payments [POST] (user)
        request: order_id, payment_method
        response: payment_id, order_id, status, paid_at
    
    - 权限矩阵
      Public: register, login, get products, get product detail
      User: create order, my orders, my order detail, pay
      Admin: create/update/delete product, view all orders
    
    - error code rule
      400 参数错误
      401 未认证/Token 无效
      403 无权限
      404 资源不存在
      409 业务冲突（如库存不足）
      500 服务内部错误
      
Phase4 Framework Design
  - 项目目录结构(Flask + SQLAlchemy)
    app/
      app/__init__.py：create_app() 工厂
      app/config.py：配置读取（沿用你已有思路）
      app/extensions.py：db、jwt、migrate 等扩展初始化
      app/models/：ORM 模型（user/product/order/payment）
      app/schemas/：请求/响应校验与序列化（可用 pydantic/marshmallow）
      app/repositories/：纯数据访问层（SQL 查询）
      app/services/：业务逻辑层（库存校验、权限判断、状态流转）
      app/api/
      app/api/auth_routes.py
      app/api/product_routes.py
      app/api/order_routes.py
      app/api/payment_routes.py
      app/api/admin_routes.py（可选，也可拆在各模块）
      app/core/
      app/core/auth.py（require_auth / require_admin）
      app/core/errors.py（业务异常定义）
      app/core/response.py（统一响应包装）
      run.py：启动入口
  
  - 结构分工明确
    - Routes：只用于解析参数，call service服务，返回响应
    - Service：只做业务逻辑
    - Repo：只做数据库CRUD
    - models：数据结构与关系（OOP）
    - schemas： 输入输出校验，字段约束
  
  - 权限设计（JWT）
    登录成功返回：access_token, token_type="bearer", expires_in
    受保护接口通过 Authorization: Bearer <token>
    admin 接口统一走 require_admin 装饰器（从 token 解析 is_admin）
    并明确：

    不从请求体传 user_id 判权限
    user_id 永远从 JWT claim 取
  
  - 请求path
    - 以“创建订单”为例，链路应是：
      Route 收参（product_id, quantity）
      Schema 校验
      Auth 获取当前 user_id
      Service 执行业务（查库存、算金额、创建订单、扣库存）
      Repository 读写 DB（事务）
      统一响应返回
  
  - response format
    - 成功：
    { "code": 0, "message": "ok", "data": {...} }
    - 失败：
    { "code": 40001, "message": "invalid params", "data": null }