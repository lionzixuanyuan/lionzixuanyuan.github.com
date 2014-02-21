---
layout: post
title: "Something About Nested Attributes"
description: ""
category: "Rails"
tags: [nested_attributes]
---
{% include JB/setup %}

在最近的工作过程中，产生了对这样的一个功能的思考：通过一个表单，在创建对象A的实例同时，创建与之相关联的B对象实例

大概构思了一下：将与B对象相关的字段放在创建A的表单中，一并提交后台后。再根据这些参数创建B的实例后，再与A的实例相关联。

    def create
      @a = A.new params[:a]
      @b = B.new params[:b]

      @b.save
      @a.b = @b
      @a.save
    end

是不是还应该加上事务操作？不用说，这种方法相当反人类。经过大神点拨，去了解了一下 Nested Attributes 相关的知识点

运用 Nested Attributes ，可以通过在创建父对象实例的同时，创建与之相关的子对象实例

在默认情况下，nested arrtibute 是关闭的，我们可以通过 #accepts_nested_attributes_for 这个类方法来启用

在启用了 nested attribute 后，会根据关联对象的名字生成一个 attribute writer 方法

在如下例子中，会生成如下的两个 attribute writer 方法：author_attributes=(attributes) 和 pages_attributes=(attributes)

    class Book < ActiveRecord::Base
      has_one :author
      has_many :pages

      accepts_nested_attributes_for :author, :pages
    end

注意：:autosave 选项会自动作用于每个启用了 #accepts_nested_attributes_for 的关联对象上

###One-to-one

假设一个 Member 模型对应一个 Avatar（阿凡达）

    class Member < ActiveRecord::Base
      has_one :avatar
      accepts_nested_attributes_for :avatar
    end

通过如下的方法可以同时创建 member 和 avatar

    params = { member: { name: 'Jack', avatar_attributes: { icon: 'smiling' } } }
    member = Member.create(params[:member])
    member.avatar.id # => 2
    member.avatar.icon # => 'smiling'

也可以通过 member 来更新与之对应的 avatar 记录

    params = { member: { avatar_attributes: { id: '2', icon: 'sad' } } }
    member.update params[:member]
    member.avatar.icon # => 'sad'

默认情况下，我们只能够设置和更新 avatar 的属性，假如需要通过 member 来删除 avatar ，就必须先设置 :allow_destroy 选项

    class Member < ActiveRecord::Base
      has_one :avatar
      accepts_nested_attributes_for :avatar, allow_destroy: true
    end

这样一来，当你在 hash 中增加  _ destroy 属性，并赋值未 true 就可以删除对应的 avatar

    member.avatar_attributes = { id: '2', _destroy: '1' }
    member.avatar.marked_for_destruction? # => true
    member.save
    member.reload.avatar # => nil

注意：avatar 只有在当 member 保存后才会真正被删除

###One-to-many

假设一个 member 对应多个 post

    class Member < ActiveRecord::Base
      has_many :posts
      accepts_nested_attributes_for :posts
    end

通过如下的形式，可以通过 member 来创建和更新对应的多个 psot 对象

没有一个不包含 id 键的 hash 都会被实例化为一个新的记录，除非这个 hash 中包含了一个 _ destroy 字段，且该字段的值为 true

    params = { member: {
      name: 'joe', posts_attributes: [
        { title: 'Kari, the awesome Ruby documentation browser!' },
        { title: 'The egalitarian assumption of the modern citizen' },
        { title: '', _destroy: '1' } # this will be ignored
      ]
    }}

    member = Member.create(params[:member])
    member.posts.length # => 2
    member.posts.first.title # => 'Kari, the awesome Ruby documentation browser!'
    member.posts.second.title # => 'The egalitarian assumption of the modern citizen'

也可以通过设置 :reject_if 来设置一些默认过滤的条件，不符合条件的 hash 将会被忽视，不进行保存

    class Member < ActiveRecord::Base
      has_many :posts
      accepts_nested_attributes_for :posts, reject_if: proc { |attributes| attributes['title'].blank? }
    end

    params = { member: {
      name: 'joe', posts_attributes: [
        { title: 'Kari, the awesome Ruby documentation browser!' },
        { title: 'The egalitarian assumption of the modern citizen' },
        { title: '' } # this will be ignored because of the :reject_if proc
      ]
    }}

    member = Member.create(params[:member])
    member.posts.length # => 2
    member.posts.first.title # => 'Kari, the awesome Ruby documentation browser!'
    member.posts.second.title # => 'The egalitarian assumption of the modern citizen'

另一种写法

    class Member < ActiveRecord::Base
      has_many :posts
      accepts_nested_attributes_for :posts, reject_if: :reject_posts

      def reject_posts(attributed)
        attributed['title'].blank?
      end
    end

假如 hash 中包含 id 字段，且能够与已经存在的记录相对应起来，那么相匹配的记录就会被修改

删除对应的 post 记录同 One-to-one 中的删除方法是相同的

我们也可以通过 hash 的形式来替代数组的形式传入参数

    Member.create(name:             'joe',
                  posts_attributes: { first:  { title: 'Foo' },
                                      second: { title: 'Bar' } })

这种方式与如下的方式是等效的

    Member.create(name:             'joe',
                  posts_attributes: [ { title: 'Foo' },
                                      { title: 'Bar' } ])

通过 hash 来传递参数时，其中的键（first、second）会被忽略
注意：这些键不能够用 +'id'+ 或者 :id 来命名

###Saving

所有针对模型的变更，包括那些被标记为删除的删除操作，都会在 parent model 保存后自动的保存或者删除

###Validating the presence of a parent model

假如需要验证，确保一个子对象实例和一个父对象实例相关联，我们可以使用 validates_presence_of  方法，如下所示

    class Member < ActiveRecord::Base
      has_many :posts, inverse_of: :member
      accepts_nested_attributes_for :posts
    end

    class Post < ActiveRecord::Base
      belongs_to :member, inverse_of: :posts
      validates_presence_of :member
    end

在 one-to-one nested associations 这种情况下，假如在指派一个父对象之前 build(in-memory) 一个子对象，那么这个子对象不会被复写

    class Member < ActiveRecord::Base
      has_one :avatar
      accepts_nested_attributes_for :avatar

      def avatar
        super || build_avatar(width: 200)
      end
    end

    member = Member.new
    member.avatar_attributes = {icon: 'sad'}
    member.avatar.width # => 200

###前台嵌套表单的实现

One-to-one

    class Product < ActiveRecord::Base
      has_one :detail
      accepts_nested_attributes_for :detail
    end

    class Detail < ActiveRecord::Base
      belongs_to :product
    end
      
    <% form_for :product do |f| %>
      <%= f.text_field :title %>
      <% f.fields_for :detail do |detail| %>
        <%= detail.text_field :manufacturer %>
      <% end %>
    <% end %>
      
    class ProductsController < ApplicationController
      def create
        @product = Product.new(params[:product])
        @product.save
      end
    end

One-to-many

    class Project < ActiveRecord::Base
      has_many :tasks
      accepts_nested_attributes_for :tasks
    end
      
    class Task < ActiveRecord::Base
      belongs_to :project
    end
      
    <% form_for @project do |f| %>
      <%= f.text_field :name %>
      <% f.fields_for :tasks do |tasks_form| %>
        <%= tasks_form.text_field :name %>
      <% end %>
    <% end %>

注意：这样操作后，会发现前台页面上，嵌套的子表单会显示空白。解决方式是在表单对应的 action 中 build 一下

    @product.build_detail

或者，我们可以直接插入 html 元素

    <input name="product[detail_attributes][manufacturer]" type="text">

以上内容均摘自互联网，希望可以帮到大家理解 Nested Attributes