定義済みインターフェイス
========================

PHP内部でのインターフェイスはユーザーランドでのそれととても良く似ています。唯一の注目に値する違いは、内部のインターフェイスにはインターフェイスが実装された際に追加で実行するハンドラーを指定できるのです。この機能は、追加の制約を強制したり、ハンドラーの置き換えをおこなったりと、様々な目的に利用できます。ユーザーランドに内部の ``compare_objects`` ハンドラーを公開している"定義済み"の ``Comparable`` インターフェイスを実装することで利用したいと思います。

インターフェイスそれ自体は下記のようになっています。

.. code-block:: php

  <?php
  interface Comparable {
      static function compare($left, $right);
  }

まずは ``MINIT`` で新しいインターフェイスを登録してみましょう。

.. code-block:: c

  zend_class_entry *comparable_ce;  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_comparable, 0, 0, 2)
      ZEND_ARG_INFO(0, obj1)
      ZEND_ARG_INFO(0, obj2)
  ZEND_END_ARG_INFO()  

  const zend_function_entry comparable_functions[] = {
      ZEND_FENTRY(
          compare, NULL, arginfo_comparable, ZEND_ACC_PUBLIC|ZEND_ACC_ABSTRACT|ZEND_ACC_STATIC
      )
      PHP_FE_END
  };  

  PHP_MINIT_FUNCTION(comparable)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "Comparable", comparable_functions);
      comparable_ce = zend_register_internal_interface(&tmp_ce TSRMLS_CC);  

      return SUCCESS;
  }

この場合、 ``PHP_ABSTRACT_ME`` は使えないことに注意してください。なぜなら、静的な抽象メソッドをサポートしていないからです。代わりに、低レベルなマクロの ``ZEND_FENTRY`` を使用します。

次に、 ``interface_gets_implemented`` ハンドラーを実装しましょう。

.. code-block:: c

  static int implement_comparable(zend_class_entry *interface, zend_class_entry *ce TSRMLS_DC)
  {
      if (ce->create_object != NULL) {
          zend_error(E_ERROR, "Comparable interface can only be used on userland classes");
      }  

      ce->create_object = comparable_create_object_override;  

      return SUCCESS;
  }  

  // MINITで
  comparable_ce->interface_gets_implemented = implement_comparable;

このインターフェイスが実装されると、 ``implement_comparable`` 関数が呼び出されます。この関数では、クラスの ``create_object`` ハンドラーをオーバーライドしています。単純化するために、前もって ``create_object`` が ``NULL`` の場合(つまり、通常のユーザーランドでのクラスの場合)にのみインターフェイスが使えるようにします。当然のことながら、任意のクラスでこれが動作する

``create_object`` のオーバーライドでは通常通りオブジェクトを作成していますが、独自の ``compare_objects`` ハンドラーをハンドラー構造体に割り当てています。

.. code-block:: c

  static zend_object_handlers comparable_handlers;  

  static zend_object_value comparable_create_object_override(zend_class_entry *ce TSRMLS_DC)
  {
      zend_object *object;
      zend_object_value retval;  

      retval = zend_objects_new(&object, ce TSRMLS_CC);
      object_properties_init(object, ce);  

      retval.handlers = &comparable_handlers;  

      return retval;
  }  

  // MINITで
  memcpy(&comparable_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
  comparable_handlers.compare_objects = comparable_compare_objects;

最後に、独自の比較ハンドラーを実装しなければなりません。このハンドラーは ``zend_interfaces.h`` で定義されている ``zend_call_method_with_2_params`` マクロを使って ``compare`` メソッドを実行します。ここで生じてくる1つの疑問はどのクラスでメソッドが実行されるのかということです。今回の実装に関しては、どちらを選択しても任意ではありますが、単に最初に渡されたオブジェクトを使うようにします。実際には、これは ``$left < $right`` では ``$left`` のクラスが使用されますが、 ``$left > $right`` の場合では ``$right`` のクラスが使用されることを意味します( ``>`` は ``<`` に変換されるため)。 

.. code-block:: c

  #include "zend_interfaces.h"  

  static int comparable_compare_objects(zval *obj1, zval *obj2 TSRMLS_DC)
  {
      zval *retval = NULL;
      int result;  

      zend_call_method_with_2_params(NULL, Z_OBJCE_P(obj1), NULL, "compare", &retval, obj1, obj2);  

      if (!retval || Z_TYPE_P(retval) == IS_NULL) {
          if (retval) {
              zval_ptr_dtor(&retval);
          }
          return zend_get_std_object_handlers()->compare_objects(obj1, obj2 TSRMLS_CC);
      }  

      convert_to_long_ex(&retval);
      result = ZEND_NORMALIZE_BOOL(Z_LVAL_P(retval));
      zval_ptr_dtor(&retval);  

      return result;
  }

上記で使用されている ``ZEND_NORMALIZE_BOOL`` マクロは返り値の整数を ``-1`` 、 ``0`` 、 ``1`` のどれかに標準化します。
必要なのはこれだけです。では新しいインターフェイスを試してみましょう(例が分かりづらければ申し訳ありません)。

.. code-block:: php

  class Point implements Comparable {
      protected $x, $y, $z;  

      public function __construct($x, $y, $z) {
          $this->x = $x; $this->y = $y; $this->z = $z;
      }  

      /* 構成要素の全てが 小さい/大きい 場合にポイントが 小さい/大きい とする */
      public static function compare($p1, $p2) {
          if ($p1->x == $p2->x && $p1->y == $p2->y && $p1->z == $p2->z) {
              return 0;
          }  

          if ($p1->x < $p2->x && $p1->y < $p2->y && $p1->z < $p2->z) {
              return -1;
          }  

          if ($p1->x > $p2->x && $p1->y > $p2->y && $p1->z > $p2->z) {
              return 1;
          }  

          // 比較出来ない
          return 1;
      }
  }  

  $p1 = new Point(1, 1, 1);
  $p2 = new Point(2, 2, 2);
  $p3 = new Point(1, 0, 2);  

  var_dump($p1 < $p2, $p1 > $p2, $p1 == $p2); // true, false, false  

  var_dump($p1 == $p1); // true  

  var_dump($p1 < $p3, $p1 > $p3, $p1 == $p3); // false, false, false
