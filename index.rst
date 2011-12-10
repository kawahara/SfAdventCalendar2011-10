.. Symfony Advent Calendar 2011 - Day 10 documentation master file, created by
   sphinx-quickstart on Sat Dec  3 15:58:47 2011.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

===============================================================
Doctrine2 Timezone 対応 - Symfony Advent Calendar 2011 - Day 10
===============================================================

:Author: Shogo Kawahara <kawahara@bucyou.net> Twitter: `@ooharabucyou`_
:Date: 2011-12-10
:License: `Creative Commons Attribution 3.0 Unported License <http://creativecommons.org/licenses/by/3.0/>`_

.. _`@ooharabucyou`: http://twitter.com/ooharabucyou

この記事は `Symfony Advent Calendar 2011 <http://atnd.org/events/22378>`_ の記事です。

`<-9日目`_ **10日目(今ここ)** 11日目->

.. _`<-9日目`: http://d.hatena.ne.jp/Kiske/20111209/1323431291
.. _`10日目->`: http://example.com

概要
====

しわっす。Symfo忍者の ooharabucyou でござる。

Doctrine2 の Cookie Books から小ネタ。

(翻訳で挫折中なので、せめて紹介)


複数のタイムゾーン対応への道
============================

複数の国で展開するようなウェブアプリケーションではタイムゾーンへの対応が必要になることがあります。

Doctrine1時代
-------------

Symfony2 ではお世話になることはないでしょうが、Doctrine1 にて timezone に対応する方法も軽く紹介。

Doctrine1 では、datatime 型の属性は、 ``Y-m-d H:i:s`` の形式で帰ってくるものでした。
即ち、タイムゾーンに対応する方法として挙げられていたのは、保存や取得の直前に、
date.timezone の値を変え、取得できたら元の date.timezone に戻すという方法でした。

`Doctrine Timestamps and User Timezones <http://kriswallsmith.net/post/136226720/doctrine-timestamps-and-user-timezones>`_

なかなか、アグレッシブな方法ですね!

Doctrine2 と DateTime
---------------------

Doctrine2 には、嬉しいことに datetime型は `\DataTime` の instance で帰ってきます。

`計算機能などに若干問題がある気がしますが <http://scriptworks.jp/blog/2011/12/how_to_avoid_pitfall_of_php_datetime/>`_
文字列を素で使うよりは便利です。

さらに、タイムゾーン情報を持つことができます。


datetimetz型
------------

Doctrine2 の DBAL には datetimetz型が用意されています。

これは、時刻情報と同時にタイムゾーン情報も記録できる優れものですが、
残念なことに、MySQL、Sqlite では利用できません。

`15. Known Vendor Issues - Doctrine DBAL v2.1.0 documantion <http://www.doctrine-project.org/docs/dbal/2.1/en/reference/known-vendor-issues.html>`_

MySQL と Doctrine2 を利用して、複数タイムゾーンに対応したような
アプリケーションを作るには一手間加える必要がありそうです。。

型を新しく定義する
------------------

データベースに保存する値について、固定のタイムゾーンになるような Type を新しく
定義してしまおうという方法があるでしょう。

Symfony2 であれば、あなたのバンドルに新たに以下のような ``UTCDateTimeType`` クラスなどをつくります。

.. code-block:: php

    <?php

  namespace YourBundle\DoctrineExtensions\DBAL\Types;

  use Doctrine\DBAL\Types\DateTimeType;
  use Doctrine\DBAL\Platforms\AbstractPlatform;
  use Doctrine\DBAL\Types\ConversionException;

  class UTCDateTimeType extends DateTimeType
  {
      static private $utc = null;

      public function convertToDatabaseValue($value, AbstractPlatform $platform)
      {
          if ($value === null) {
              return null;
          }

          if (is_null(self::$utc)) {
              self::$utc = new \DateTimeZone('UTC');
          }

          $value->setTimeZone(self::$utc);

          return $value->format($platform->getDateTimeFormatString());
      }

      public function convertToPHPValue($value, AbstractPlatform $platform)
      {
          if ($value === null) {
              return null;
          }

          if (is_null(self::$utc)) {
              self::$utc = new \DateTimeZone('UTC');
          }

          $val = \DateTime::createFromFormat($platform->getDateTimeFormatString(), $value, self::$utc);

          if (!$val) {
              throw ConversionException::conversionFailed($value, $this->getName());
          }

          return $val;
      }
  }

この型は、データベースに保存するときは、必ず UTC で保存するというものです。
取得時は、UTCのタイムゾーンの日時として、 ``\DateTime`` のインスタンスを作ります。
あとの動きは、通常の ``datetime`` と同様です。

そして、それを使えるようにしてやるだけです。

``app/config/config.yml`` の ``doctrine.dbal.types`` の値をいじくってやります。

::

  doctrine:
      dbal:
          driver:   %database_driver%
          (省略)
          charset:  UTF8
          types:
              utcdatetime: YourBundle\DoctrineExtensions\DBAL\Types\UTCDateTimeType #追加

これだけで、 ``utcdatetime`` 型が使えるようになりますぞ。

もちろん、表示前に ``DateTime::setTimeZone()`` で、利用者側で見たいタイムゾーンを
設定してやる必要があるのは注意。

また明日
========

明日は、 `@iteman <http://twitter.com/#!/iteman>`_ マスターセンセーの日でござる。

参考資料
========

- `Working with DateTime Instances <http://www.doctrine-project.org/docs/orm/2.1/en/cookbook/working-with-datetime.html>`_
- `Timestampable behavior extension for Doctrine 2 <https://github.com/l3pp4rd/DoctrineExtensions/blob/master/doc/timestampable.md>`_
