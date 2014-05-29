---
layout: post
title:  "How we migrated from globalize to hstore_translate"
date:   2014-05-24 21:15:19
categories: rails
metas: [i18n, globalize, hstore-translate, rails, postgresql]
---

As part of an international company ([Leadformance][leadformance]), we soon had to deal with some translated contents.
At some point, we wanted to offer our clients to customize the content for several languages.
This means translations stored in the database.

## Separated Translations Tables

Back in 2011, when we had to implement this feature, it was obvious that we will go with [`globalize`][globalize] (we will go with [`globalize-accessors`][globalize-accessors] as well to ease the thing).

This was working pretty fine.

Here is how the model looked like:

{% highlight ruby %}
  class Post < ActiveRecord::Base
    translates :title, :text
    globalize_accessors :locales => [:en, :fr], :attributes => [:title, :text]
  end
{% endhighlight %}

### Create

To create a record we just had to do:

{% highlight ruby %}
  Post.create(title: "Title", text: "Text")
   (1.0ms)  BEGIN
  SQL (4.2ms)  INSERT INTO "posts" ("created_at", "updated_at") VALUES ($1, $2) RETURNING "id"  [["created_at", "2014-05-25 17:31:44.589125"], ["updated_at", "2014-05-25 17:31:44.589125"]]
  SQL (2.2ms)  INSERT INTO "post_translations" ("created_at", "locale", "post_id", "text", "title", "updated_at") VALUES ($1, $2, $3, $4, $5, $6) RETURNING "id"  [["created_at", "2014-05-25 17:31:44.612382"], ["locale", "en"], ["post_id", 1], ["text", "Text"], ["title", "Title"], ["updated_at", "2014-05-25 17:31:44.612382"]]
   (1.1ms)  COMMIT
=> #<Post id: 1, created_at: "2014-05-25 17:31:44", updated_at: "2014-05-25 17:31:44">
{% endhighlight %}

Or, if we wanted to create several languages in a row:

{% highlight ruby %}
  Post.create(title_fr: "Titre", text_fr: "Texte", title_en: "Title", text_en: "Text")
   (3.8ms)  BEGIN
  SQL (3.8ms)  INSERT INTO "posts" ("created_at", "updated_at") VALUES ($1, $2) RETURNING "id"  [["created_at", "2014-05-25 17:39:42.736275"], ["updated_at", "2014-05-25 17:39:42.736275"]]
  SQL (3.2ms)  INSERT INTO "post_translations" ("created_at", "locale", "post_id", "text", "title", "updated_at") VALUES ($1, $2, $3, $4, $5, $6) RETURNING "id"  [["created_at", "2014-05-25 17:39:42.760307"], ["locale", "fr"], ["post_id", 2], ["text", "Texte"], ["title", "Titre"], ["updated_at", "2014-05-25 17:39:42.760307"]]
  SQL (1.4ms)  INSERT INTO "post_translations" ("created_at", "locale", "post_id", "text", "title", "updated_at") VALUES ($1, $2, $3, $4, $5, $6) RETURNING "id"  [["created_at", "2014-05-25 17:39:42.767345"], ["locale", "en"], ["post_id", 2], ["text", "Text"], ["title", "Title"], ["updated_at", "2014-05-25 17:39:42.767345"]]
   (1.2ms)  COMMIT
=> #<Post id: 2, created_at: "2014-05-25 17:39:42", updated_at: "2014-05-25 17:39:42">
{% endhighlight %}

(Notice that this was generating 3 `INSERT`s for a single record)

### Update

If we wanted to update the translations, the gems handled that for us as well:

{% highlight ruby %}
  post.title_fr = "Mon titre français"
=> "Mon titre français"
  post.save
   (3.6ms)  BEGIN
  SQL (2.0ms)  UPDATE "posts" SET "updated_at" = $1 WHERE "posts"."id" = 2  [["updated_at", "2014-05-25 17:56:34.915778"]]
  SQL (3.4ms)  UPDATE "post_translations" SET "title" = $1, "updated_at" = $2 WHERE "post_translations"."id" = 2  [["title", "Mon titre français"], ["updated_at", "2014-05-25 17:56:34.957760"]]
   (1.4ms)  COMMIT
=> true
{% endhighlight %}

### `include` and `joins`

To avoid the `n+1` queries, we had to put the `includes`:

{% highlight ruby %}
  Post.includes(:translations)
  Post Load (1.4ms)  SELECT "posts".* FROM "posts"
  Post::Translation Load (3.0ms)  SELECT "post_translations".* FROM "post_translations"  WHERE "post_translations"."post_id" IN (2, 3)
=> #<ActiveRecord::Relation [#<Post id: 2, created_at: "2014-05-25 17:31:44", updated_at: "2014-05-25 17:31:44">, #<Post id: 3, created_at: "2014-05-25 17:39:42", updated_at: "2014-05-25 17:39:42">]>
{% endhighlight %}

Sometimes, we would also wanted to perform some queries directly tied to the translations.
For example, to retrieve all `Post`s having a `title` translated in french, we would have done this:

{% highlight ruby %}
  Post.includes(:translations).
    joins(:translations).
    where(post_translations: { locale: "fr" }).
    where("post_translations.title IS NOT NULL")
  SQL (1.6ms)  SELECT "posts"."id" AS t0_r0, "posts"."created_at" AS t0_r1, "posts"."updated_at" AS t0_r2, "post_translations"."id" AS t1_r0, "post_translations"."post_id" AS t1_r1, "post_translations"."locale" AS t1_r2, "post_translations"."created_at" AS t1_r3, "post_translations"."updated_at" AS t1_r4, "post_translations"."title" AS t1_r5, "post_translations"."text" AS t1_r6 FROM "posts" INNER JOIN "post_translations" ON "post_translations"."post_id" = "posts"."id" WHERE "post_translations"."locale" = 'fr' AND (post_translations.title IS NOT NULL)
=> #<ActiveRecord::Relation [#<Post id: 2, created_at: "2014-05-25 17:39:42", updated_at: "2014-05-25 17:39:42">]>
{% endhighlight %}

## `hstore` to the rescue

Though, as we are using PostgreSQL, we thought we weren't using the real potential of it and we could use something even clearer. I'm talking about `hstore`.

It makes it clearer simply because your fields are now in the same table than your other data.

As per the [documentation][hstore-doc], the `hstore` data type is for storing sets of key/value pairs within a single PostgreSQL value.

This is exactly what we need right?

For this purpose, we're gonna use [`hstore-translate`][hstore-translate] written by the awesome [Rob Worley][rob-worley].

The great thing with `hstore-translate` is that it makes the thing really easy to switch from `globalize` + `globalize-accessors` to it as there are sharing the same API.
That also means that you cannot use `globalize` AND `hstore-translate` at the same time.

The model now looks like this:

{% highlight ruby %}
class Post < ActiveRecord::Base
  translates :title, :text
end
{% endhighlight %}

We don't need to setup the accessible locales in the model as it is part of `hstore-translate` itself.

### Create

To create the records, it works exactly the same way than previously but:

{% highlight ruby %}
  Post.create(title: "Title", text: "Text")
  (3.0ms)  BEGIN
 SQL (1.7ms)  INSERT INTO "posts" ("created_at", "text_translations", "title_translations", "updated_at") VALUES ($1, $2, $3, $4) RETURNING "id"  [["created_at", "2014-05-25 18:51:07.848169"], ["text_translations", "\"en\"=>\"Text\""], ["title_translations", "\"en\"=>\"Title\""], ["updated_at", "2014-05-25 18:51:07.848169"]]
  (1.1ms)  COMMIT
=> #<Post id: 1, title_translations: {"en"=>"Title"}, text_translations: {"en"=>"Text"}, created_at: "2014-05-25 18:51:07", updated_at: "2014-05-25 18:51:07">

  Post.create(title_en: "Title", text_en: "Text", title_fr: "Titre", text_fr: "Texte")
   (1.0ms)  BEGIN
  SQL (2.2ms)  INSERT INTO "posts" ("created_at", "text_translations", "title_translations", "updated_at") VALUES ($1, $2, $3, $4) RETURNING "id"  [["created_at", "2014-05-25 18:48:50.470176"], ["text_translations", "\"en\"=>\"Text\",\"fr\"=>\"Texte\""], ["title_translations", "\"en\"=>\"Title\",\"fr\"=>\"Titre\""], ["updated_at", "2014-05-25 18:48:50.470176"]]
   (1.2ms)  COMMIT
=> #<Post id: 2, title_translations: {"en"=>"Title", "fr"=>"Titre"}, text_translations: {"en"=>"Text", "fr"=>"Texte"}, created_at: "2014-05-25 18:48:50", updated_at: "2014-05-25 18:48:50">
{% endhighlight %}

### Update

{% highlight ruby %}
 post.title_fr = "Titre Français"
=> "Titre Français"
 post.save
   (3.5ms)  BEGIN
  SQL (3.8ms)  UPDATE "posts" SET "title_translations" = $1, "updated_at" = $2 WHERE "post_hstores"."id" = 2  [["title_translations", "\"en\"=>\"Title\",\"fr\"=>\"Titre Français\""], ["updated_at", "2014-05-25 20:05:17.925179"]]
   (1.0ms)  COMMIT
=> true
{% endhighlight %}

It's now faster as it's tied to only one table, but it's also easier to query:

{% highlight ruby %}
  Post.where("defined(title_translations, ?)", 'fr')
  Post Load (1.4ms)  SELECT "posts".* FROM "posts"  WHERE (defined(title_translations, 'fr'))
=> #<ActiveRecord::Relation [#<Post id: 2, title_translations: {"en"=>"Title", "fr"=>"Titre"}, text_translations: {"en"=>"Text", "fr"=>"Texte"}, created_at: "2014-05-25 18:48:50", updated_at: "2014-05-25 18:48:50">]>
{% endhighlight %}

Or even (provided by the gem):

{% highlight ruby %}
Post.with_title_translation("Titre Français", "fr")
  Post Load (3.1ms)  SELECT "posts".* FROM "posts"  WHERE ("title_translations" @> hstore('fr', 'Titre Français'))
=> #<ActiveRecord::Relation [#<Post id: 2, title_translations: {"en"=>"Title", "fr"=>"Titre Français"}, text_translations: nil, created_at: "2014-05-25 19:42:32", updated_at: "2014-05-25 20:13:54">]>

{% endhighlight %}

### Bonus Point

We can do something like this as well:

{% highlight ruby %}
  Post.where("defined(title_translations, ?)", 'fr').
    pluck("title_translations -> 'fr'")
   (1.5ms)  SELECT title_translations -> 'fr' FROM "posts"  WHERE (defined(title_translations, 'fr'))
=> ["Titre"]
{% endhighlight %}

## Migration

The migration has been really painless thanks to the shared API.

I've issued a [Pull Request][improve-globalize-compatibility-for-hstore-translate] to ease the migration based on what we encountered ourselves.

If you are coming from `globalize`, you'll probably have to migrate your data.
If so, here is an example of what we used to make it happen.

{% highlight ruby %}
execute <<-SQL
  UPDATE posts
  SET
    title_translations = translations.title
  FROM (
      SELECT
        hstore(
          array_agg(locale ORDER BY locale),
          array_agg(post_translations.title ORDER BY locale)
        ) AS title,
        post_id
    FROM post_translations
    GROUP BY post_id
  ) translations
  WHERE translations.post_id = posts.id;
SQL
{% endhighlight %}

## Conclusion

We are really happy about this choice to change our translation mechanism.
It reduces the overhead of having the data related to a single record shared across several tables.

We are now able to provide to our clients more fields to translate and this is possible without impacting the performances.

Thanks for reading.

[leadformance]: http://www.leadformance.com
[globalize]: https://github.com/globalize/globalize
[globalize-accessors]: https://github.com/globalize/globalize-accessors
[hstore-doc]: http://www.postgresql.org/docs/9.4/static/hstore.html
[hstore-translate]: https://github.com/robworley/hstore_translate
[rob-worley]: https://github.com/robworley
[improve-globalize-compatibility-for-hstore-translate]: https://github.com/robworley/hstore_translate/pull/28
