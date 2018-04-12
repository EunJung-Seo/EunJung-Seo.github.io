---
title: "utf8에서 utf8mb4로 character set 변경하기"
date: "2018-04-11T17:30:51.851Z"
layout: post
draft: false
path: "/posts/migrate-utf8-to-utf8mb4-in-rails/"
category: "Rails"
tags:
  - "Web Development"
description: "이전 회사에서 백엔드 개발자로 일하는 동안 가장 빨리 해결하고 싶었던 문제 중 하나가 바로 이모지 저장이다. 슬랙이나 깃헙, SNS 등에서 이모지를 많이 사용하고 있었지만 정작 내가 만드는 서비스에서는 이모지를 사용할 수 없다는 게 꽤나 마음에 걸렸다. 이모지와 관련한 문의들은 대부분 다음과 같다."
---

이전 회사에서 백엔드 개발자로 일하는 동안 가장 빨리 해결하고 싶었던 문제 중 하나가 바로 이모지 저장이다. 슬랙이나 깃헙, SNS 등에서 이모지를 많이 사용하고 있었지만 정작 내가 만드는 서비스에서는 이모지를 사용할 수 없다는 게 꽤나 마음에 걸렸다. 이모지와 관련한 문의들은 대부분 다음과 같다.

> 글을 작성했는데 저장이 안돼요!
> 저장하기를 누르고 나니 작성한 글이 다 사라졌어요!

사람들이 이모지를 자주 사용하게 되면서 위와 같은 문의도 점차 증가하기 시작했다. 처음엔 글 작성시의 validation 오류나 사용자의 실수라고 생각했다. 하지만 조금 더 들여다보니(거의 2년 전이라 어떻게 발견했는지 기억이 나질 않는다) 문제는 이모지에 있다는 것을 알게 되었다. DB로 MySQL 사용하고 있었는데 대부분의 character set이 utf8으로 되어있어 4바이트인 이모지를 저장할 수 없었던 것이다. 이모지를 사용하기 직전의 내용까지만 저장되고 그 뒤의 내용은 다 날아가 버렸다.
꽤나 긴 분량의 글을 작성하고 날려버린 사용자들은 운영팀에 도움을 청했고 나는 그때마다 DB 스냅샷을 찾아 글을 복구해주곤 했다. 하지만 언제까지 이 일을 반복할 순 없는 노릇! 문제 발견 후 1년이 지나서야(…) 인코딩 변경에 들어갔다. 

> (참고로 Rails는 3.2, MySQL은 5.6.34 버전이다.)


블로그 포스트를 예로 들어보자. posts 테이블에는 text 데이터 타입의 body와 string 데이터 타입의 title이 있다. 우리는 body를 utf8에서 utf8mb4로 변경하려 한다. 아래와 같은 migration 파일을 작성한 뒤, `rake db:migrate`를 해주자.
```ruby
class ConvertDatabaseToUtf8mb4 < ActiveRecord::Migration
  def change
    execute 'ALTER TABLE posts '\
    'CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
    execute 'ALTER TABLE posts '\
    'MODIFY `body` TEXT CHARACTER SET utf8mb4 utf8mb4_unicode_ci;'
  end
end
```
그러나 다음과 같은 에러가 난다!
```ruby
Mysql2::Error: Specified key was too long; max key length is 767 byte
```
string 데이터 타입의 title 컬럼은 255자까지 저장 가능한데 이는 한 글자 당 3바이트인 utf8기준이다. 글자 당 4바이트인 utf8mb4로 변경되면 191자까지 가능하다. string limit을 191로 설정해 주면 에러를 피할 수 있다.
change 함수 안에 `change_column :posts, :title, :string, limit: 191`를 한 줄 추가해주고 다시 `rake db:migrate`를 해준다.

여기까지 문제없이 진행했다면 DB에 이모지를 쓰고 불러올 수 있다. 

만약 이모지가 포함된 텍스트를 json으로 렌더링 해줘야 한다면 한 가지 작업을 더 해줘야 한다. 내 경우는 Rails를 API로 사용하고 있어 json 렌더링이 필수였다. json으로 데이터를 변환하면 DB에 잘 저장되었어도 유니코드가 잘려 글자가 깨져버렸다. 검색해보니 나와 같은 문제를 겪는 사람들이 꽤 됐고 일반적으로 사용되는 monkey patch가 있어 나도 동일하게 적용해 주었다.

```ruby
# config/initializer/active_support_encoding.rb
# patch for utf8mb4 encoding
module ActiveSupport::JSON::Encoding
  class << self
    def escape(string)
      if string.respond_to?(:force_encoding)
        string = string.encode(::Encoding::UTF_8, undef: :replace).force_encoding(::Encoding::BINARY)
      end
      json = string.gsub(escape_regex) { |s| ESCAPED_CHARS[s] }
      json = %("#{json}")
      json.force_encoding(::Encoding::UTF_8) if json.respond_to? (:force_encoding)
      json  
    end
  end
end
```
이렇게 해주면 json으로도 문제 없이 이모지를 주고 받을 수 있다.
아래는 내가 참고한 페이지들이다.

- [Emoji and Rails JSON output issue - Dan Sosedoff](https://sosedoff.com/2012/02/18/emoji-and-rails.html)
- [Emoji range encoded in JSON mangled · Issue #3727 · rails/rails · GitHub](https://github.com/rails/rails/issues/3727)