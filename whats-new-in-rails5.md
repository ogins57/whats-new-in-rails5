
class: center, middle

# Rails 5 の新機能

2016/04/07 荻野 浩史

---

# アジェンダ

1. development 環境でのキャッシュ ON／OFF が簡単になりました
2. belongs_to のデフォルトが required: true になりました
3. routes に検索オプションが導入されました
4. redirect_to :back が非推奨になりました
5. migration ファイルの記述が変更されました
6. ネストされたパラメータのフィルタリングが可能になりました
7. left_outer_join が導入されました

---

### 1. development 環境でのキャッシュ ON／OFF が簡単になりました

以前は、たとえばローカル環境で memcached の挙動を確認したいときは `config/environments/development.rb` を編集する必要がありました。

```ruby:config/environments/development.rb
config.action_controller.perform_caching = true
config.cache_store = :mem_cache_store
```

---
 
### 1. development 環境でのキャッシュ ON／OFF が簡単になりました

Rails 5 では `rails dev:cache` というコマンドが導入され、そのコマンドの実行でキャッシュの ON / OFF を切り替えられるようになりました。

（１回実行すればキャッシュを ON に）
```bash
$ rails dev:cache
Development mode is now being cached.
```

（もう１回実行すれば OFF にできる）

```bash
$ rails dev:cache
Development mode is no longer being cached.
```

---

### 1. development 環境でのキャッシュ ON／OFF が簡単になりました

仕組みは単純、`tmp/caching-dev.txt` があれば mem_cache_store、なければ null_store を利用しています。

`config/environments/development.rb`

```ruby:config/environments/development.rb
if Rails.root.join('tmp/caching-dev.txt').exist?
  config.action_controller.perform_caching = true
  config.static_cache_control = "public, max-age=172800"
  config.cache_store = :mem_cache_store
else
  config.action_controller.perform_caching = false
  config.cache_store = :null_store
end
```

---

### 1. development 環境でのキャッシュ ON／OFF が簡単になりました

また、`rails dev:cache` を叩くたびに `tmp/caching-dev.txt` を作ったり消したりしています。

```ruby:dev
def dev_cache
  if File.exist? 'tmp/caching-dev.txt'
    File.delete 'tmp/caching-dev.txt'
    puts 'Development mode is no longer being cached.'
  else
    FileUtils.touch 'tmp/caching-dev.txt'
    puts 'Development mode is now being cached.'
  end

  FileUtils.touch 'tmp/restart.txt'
end
```


---

### 2. belongs_to のデフォルトが required: true になりました

Rails 5 では belongs_to で指定したアソシエーションのインスタンスがデフォルトで必須になりました。
そのため、例えば下記の場合、User のインスタンス無しに Post をインスタンス化することは出来なくなりました。

```ruby:x
class User < ApplicationRecord
end

class Post < ApplicationRecord
  belongs_to :user
end

post = Post.create(title: 'Hi')
=> <Post id: nil, title: "Hi", user_id: nil, created_at: nil, updated_at: nil>

post.errors.full_messages.to_sentence
=> "User must exist"
```

---

### 2. belongs_to のデフォルトが required: true になりました

つまりはこういうことです。

```ruby:x
class User < ApplicationRecord
end

class Post < ApplicationRecord
  belongs_to :user, required: true  #「required: true」が指定されている
end

post = Post.create(title: 'Hi')
=> <Post id: nil, title: "Hi", user_id: nil, created_at: nil, updated_at: nil>

post.errors.full_messages.to_sentence
=> "User must exist"
```

---

### 2. belongs_to のデフォルトが required: true になりました

これまでと同じように belongs_to で指定したアソシエーションのインスタンスが無くても動作するようにするためには `optional: true` を指定します。

```ruby:x
class Post < ApplicationRecord
  belongs_to :user, optional: true  #「optional: true」を指定します
end

post = Post.create(title: 'Hi')
=> <Post id: 2, title: "Hi", user_id: nil>
```

---

### 2. belongs_to のデフォルトが required: true になりました

モデルごとではなくシステム全体に適用することも可能です。

`config/initializers/active_record_belongs_to_required_by_default.rb`

```ruby:x
Rails.application.config.active_record.belongs_to_required_by_default = false
```

---

### 3. routes に検索オプションが導入されました

今までは routes ＋ grep で検索してましたよね？
Rails 5 からはその必要がなくなりました。

```bash
$ rake routes | grep products
Prefix       Verb   URI Pattern                   Controller#Action
products      GET    /products(.:format)           products#index
              POST   /products(.:format)           products#create
```


---

### 3. routes に検索オプションが導入されました

Controller を検索する場合は `-c` オプションを付けましょう。

```bash
# Search for Controller name
$ rails routes -c users
       Prefix Verb   URI Pattern                   Controller#Action
wishlist_user GET    /users/:id/wishlist(.:format) users#wishlist
        users GET    /users(.:format)              users#index
              POST   /users(.:format)              users#create
```

```bash
# Search for namespaced Controller name.
$ rails routes -c admin/users
         Prefix Verb   URI Pattern                     Controller#Action
    admin_users GET    /admin/users(.:format)          admin/users#index
                POST   /admin/users(.:format)          admin/users#create
```

```bash
# Search for namespaced Controller name.
$ rails routes -c Admin::UsersController
         Prefix Verb   URI Pattern                     Controller#Action
    admin_users GET    /admin/users(.:format)          admin/users#index
                POST   /admin/users(.:format)          admin/users#create
```

---

### 3. routes に検索オプションが導入されました

パターンマッチで検索したいときは `-g` オプションを付けましょう。

```bash
# Search with pattern
$ rails routes -g wishlist
       Prefix Verb URI Pattern                   Controller#Action
wishlist_user GET  /users/:id/wishlist(.:format) users#wishlist
```

```bash
# Search with HTTP Verb
$ rails routes -g POST
    Prefix Verb URI Pattern            Controller#Action
           POST /users(.:format)       users#create
           POST /admin/users(.:format) admin/users#create
           POST /products(.:format)    products#create
```

```bash
# Search with URI pattern
$ rails routes -g admin
       Prefix Verb   URI Pattern                     Controller#Action
  admin_users GET    /admin/users(.:format)          admin/users#index
              POST   /admin/users(.:format)          admin/users#create
```

---

### 4. redirect_to :back が非推奨になりました

Rails 4 で前ページに戻るときに `redirect_to :back` を利用してました。
これは `HTTP_REFERER` が設定されてない場合に `ActionController::RedirectBackError` が発行されるため、下記のようにコーディングする必要がありましたね。

```ruby
class PostsController < ApplicationController
  rescue_from ActionController::RedirectBackError, with: :redirect_to_default

  def publish
    post = Post.find params[:id]
    post.publish!
    redirect_to :back
  end

  private

  def redirect_to_default
    redirect_to root_path
  end
end
```

---

### 4. redirect_to :back が非推奨になりました

Rails 5 では `redirect_to :back` が非推奨となり、新たに `redirect_back` というメソッドが導入されました。
このメソッドでは `HTTP_REFERER` が存在しない場合に備えて `fallback_location` というオプションが利用できます。

```ruby
class PostsController < ApplicationController

  def publish
    post = Post.find params[:id]
    post.publish!

    redirect_back(fallback_location: root_path)
  end
end
```

---

### 5. migration ファイルの記述が変更されました

```bash
$ rails g model User name:string
```

`timestamps` から `null: false` が削除されました。
と言っても NOT NULL 制約が外されたわけではなく、デフォルトでその制約が盛り込まれるようになりました。（作成されるスキーマは以前と同じになります）


```ruby
class CreateUsers < ActiveRecord::Migration[5.0]
  def change
    create_table :users do |t|
      t.string :name

      # t.timestamps null: false  # 以前はこうだった
      t.timestamps
    end
  end
end
```

---

### 5. migration ファイルの記述が変更されました

```bash
$ rails g model Task user:references
```

`references` から `index: true` が削除されました。
こちらも index が張られなくなったわけではなく、デフォルトで index が張られるようになりました。（作成されるスキーマは以前と同じになります）

```ruby
class CreateTasks < ActiveRecord::Migration[5.0]
  def change
    create_table :tasks do |t|
      # t.references :user, index: true, foreign_key: true  # 以前はこうだった
      t.references :user, foreign_key: true

      t.timestamps
    end
  end
end
```

---

### 5. migration ファイルの記述が変更されました

すでにお気付きかもしれませんがベースクラスが `ActiveRecord::Migration` から `ActiveRecord::Migration[5.0]` に変更されました。

```ruby
# class CreateTasks < ActiveRecord::Migration  # 以前はこうだった
class CreateTasks < ActiveRecord::Migration[5.0]
end
```

これは Rails 5 用のマイグレーション処理を実行するためのもので、以前のバージョンに合わせた処理を実行したい場合は下記のようにします。

```ruby
class CreateTasks < ActiveRecord::Migration[4.2]
end
```

---

### 6. ネストされたパラメータのフィルタリングが可能になりました

これまではパスワードをフィルタリングしたい場合、以下のように定義していました。
（こうすることでログファイル上では `[FILTERED]` と表示されるようになってました。）

`config/initializers/filter_parameter_logging.rb`

```ruby:config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [:password]
```


---

### 6. ネストされたパラメータのフィルタリングが可能になりました

この機能は「ネストしたデータを指定できない」という問題を抱えていました。
たとえば下記のようにクレジットカード番号をフィルタリングしたい場合、同時に user.color.code もフィルタリングされてしまっていました。

```ruby:config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += ['code']
```

```ruby
{ credit_card: { number: '123456789', code: [FILTERD] } }
{ user_preference: { color: { name: 'Grey', code: [FILTERD] } } }

# user.color.code もフィルタリングされてしまう
```

---

### 6. ネストされたパラメータのフィルタリングが可能になりました

Rails 5 ではネストしたデータを指定できるようになったため、下記のように設定すればクレジットカード番号だけフィルタリングすることが可能になりました。

```ruby:config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += ['credit_card.code']
```

```ruby
{ credit_card: { number: '123456789', code: [FILTERD] } }
{ user_preference: { color: { name: 'Grey', code: '999999' } } }

# user.color.code はフィルタリングされない
```

---

### 7. left_outer_join が導入されました

今まで LEFT OUTER JOIN したいときはわざわざ自分で書いてましたよね？
そんな時代がようやく終焉を迎えました。

```ruby
authors = Author.join('LEFT OUTER JOIN "posts" ON "posts"."author_id" = "authors"."id"')
                .uniq
                .select("authors.*, COUNT(posts.*) as posts_count")
                .group("authors.id")
```

---

### 7. left_outer_join が導入されました

これからは left_outer_joins を使いましょう。

```ruby
authors = Author.left_outer_joins(:posts)
                .uniq
                .select("authors.*, COUNT(posts.*) as posts_count")
                .group("authors.id")
```

