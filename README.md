# AngularJS スタイルガイド

*チーム開発のための AngularJS スタイルガイド by [@toddmotto](//twitter.com/toddmotto)*

[講演](https://speakerdeck.com/toddmotto)やチーム開発での [Angular](//angularjs.org) の経験により作成した AngularJS アプリケーションのシンタックスや構造についてのスタイルガイド。

> See the [original article](http://toddmotto.com/opinionated-angular-js-styleguide-for-teams)

## 目次

  1. [Modules](#modules)
  1. [Controllers](#controllers)
  1. [Services](#services)
  1. [Factory](#factory)
  1. [Directives](#directives)
  1. [Filters](#filters)
  1. [Routing resolves](#routing-resolves)
  1. [Publish and subscribe events](#publish-and-subscribe-events)
  1. [Angular wrapper references](#angular-wrapper-references)
  1. [Comment standards](#comment-standards)
  1. [Minification and annotation](#minification-and-annotation)

## Modules

  - **定義**: 変数を使わずに setter / getter で module を定義

    ```javascript
    // bad
    var app = angular.module('app', []);
    app.controller();
    app.factory();

    // good
    angular
      .module('app', [])
      .controller()
      .factory();
    ```

  - 注釈: `angular.module('app', []);` を setter、`angular.module('app');` を getter として使う。setter で module を定義し、他のインスタンスからは getter でその module を取得して利用する。

  - **メソッド**: コールバックとして記述せず、function を定義してメソッドに渡す

    ```javascript
    // bad
    angular
      .module('app', [])
      .controller('MainCtrl', function MainCtrl () {

      })
      .service('SomeService', function SomeService () {

      });

    // good
    function MainCtrl () {

    }
    function SomeService () {

    }
    angular
      .module('app', [])
      .controller('MainCtrl', MainCtrl)
      .service('SomeService', SomeService);
    ```

  - コードのネストが深くなることを抑え、可読性を高める
  
  - **IIFE（イッフィー：即時関数式）スコープ**: Angular に渡す function の定義でグローバルスコープを汚染することを避けるため、複数ファイルを連結（concatenate）するビルドタスクで IIFE 内にラップする
  
    ```javascript
    (function () {

      angular
        .module('app', []);
      
      // MainCtrl.js
      function MainCtrl () {

      }
      
      angular
        .module('app')
        .controller('MainCtrl', MainCtrl);
      
      // SomeService.js
      function SomeService () {

      }
      
      angular
        .module('app')
        .service('SomeService', SomeService);
        
      // ...
        
    })();
    ```


**[Back to top](#table-of-contents)**

## Controllers

  - **controllerAs 構文**: Controller はクラスであるため、常に `controllerAs` 構文を利用する

    ```html
    <!-- bad -->
    <div ng-controller="MainCtrl">
      {{ someObject }}
    </div>

    <!-- good -->
    <div ng-controller="MainCtrl as main">
      {{ main.someObject }}
    </div>
    ```

  - DOM において controller につき変数を定義することで、ネストされた controller で `$parent` の呼び出しを避ける

  - `controllerAs` 構文では `$scope` にバインドされる controller 内で `this` を利用する

    ```javascript
    // bad
    function MainCtrl ($scope) {
      $scope.someObject = {};
      $scope.doSomething = function () {

      };
    }

    // good
    function MainCtrl () {
      this.someObject = {};
      this.doSomething = function () {

      };
    }
    ```

  - `$emit`、`$broadcast`、`$on` や `$watch` を利用する場合など、必要なケースに限って `controllerAs` 内で `$scope` を利用する

  - **継承**: controller クラスを拡張する場合は prototype 継承を利用する

    ```javascript
    function BaseCtrl () {
      this.doSomething = function () {

      };
    }
    BaseCtrl.prototype.someObject = {};
    BaseCtrl.prototype.sharedSomething = function () {

    };

    AnotherCtrl.prototype = Object.create(BaseCtrl.prototype);

    function AnotherCtrl () {
      this.anotherSomething = function () {

      };
    }
    ```

  - `Object.create` をレガシーブラウザでもサポートするためには polyfill を利用する

  - **Zero-logic**: controller 内にはロジックを無くし, service に委譲する

    ```javascript
    // bad
    function MainCtrl () {
      this.doSomething = function () {

      };
    }

    // good
    function MainCtrl (SomeService) {
      this.doSomething = SomeService.doSomething;
    }
    ```

  - "skinny controller, fat service" を念頭に置く

**[Back to top](#table-of-contents)**

## Services

  - service はクラスで、`new` でインスタンス化し、パブリックなメソッドと変数には `this` を利用する

    ```javascript
    function SomeService () {
      this.someMethod = function () {

      };
    }
    ```

**[Back to top](#table-of-contents)**

## Factory

  - **シングルトン**: factory はシングルトンで、primitive binding issues（子スコープの変更を親スコープで検知できなくなる問題で、ng-model で '.' を使うようにすることで回避可能な問題）を避けるために factory 内のホストオブジェクトを返す

    ```javascript
    // bad
    function AnotherService () {
      var someValue = '';
      var someMethod = function () {

      };
      return {
        someValue: someValue,
        someMethod: someMethod
      };
    }

    // good
    function AnotherService () {
      var AnotherService = {};
      AnotherService.someValue = '';
      AnotherService.someMethod = function () {

      };
      return AnotherService;
    }
    ```

  - このバインディングであればホストオブジェクト越しにミラーされる。"Revealing Module Pattern" では primitive の値は更新されない。

**[Back to top](#table-of-contents)**

## Directives

  - **restrict**: 独自 directive には `custom element` と `custom attribute` のみ利用する（`{ restrict: 'EA' }`）

    ```html
    <!-- bad -->

    <!-- directive: my-directive -->
    <div class="my-directive"></div>

    <!-- good -->

    <my-directive></my-directive>
    <div my-directive></div>
    ```

  - コメントとクラス名での宣言は混乱しやすいため使うべきでない。コメントでの宣言は古いバージョンの IE で動作せず、属性での宣言が最も安全である。

  - **template**: テンプレートをすっきりさせるために `Array.join('')` を利用する

    ```javascript
    // bad
    function someDirective () {
      return {
        template: '<div class="some-directive">' +
          '<h1>My directive</h1>' +
        '</div>'
      };
    }

    // good
    function someDirective () {
      return {
        template: [
          '<div class="some-directive">',
            '<h1>My directive</h1>',
          '</div>'
        ].join('')
      };
    }
    ```

  - **DOM 操作**: directive 内でのみ行い、controller / service では決して行わない

    ```javascript
    // bad
    function UploadCtrl () {
      $('.dragzone').on('dragend', function () {
        // handle drop functionality
      });
    }
    angular
      .module('app')
      .controller('UploadCtrl', UploadCtrl);

    // good
    function dragUpload () {
      return {
        restrict: 'EA',
        link: function (scope, element, attrs) {
          element.on('dragend', function () {
            // handle drop functionality
          });
        }
      };
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);
    ```

  - **命名規約**: 将来、標準の directive と名前が衝突しないよう、独自 directive に `ng-*` を使わない

    ```javascript
    // bad
    // <div ng-upload></div>
    function ngUpload () {
      return {};
    }
    angular
      .module('app')
      .directive('ngUpload', ngUpload);

    // good
    // <div drag-upload></div>
    function dragUpload () {
      return {};
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);
    ```

  - directive と filter を先頭文字を小文字で命名する。これは、Angular が `camelCase` をハイフンつなぎにする命名規約によるもので、つまり `dragUpload` が要素で使われるときには `<div drag-upload></div>` となる。

  - **controllerAs**: directive 内でも `controllerAs` 構文を利用する

    ```javascript
    // bad
    function dragUpload () {
      return {
        controller: function ($scope) {

        }
      };
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);

    // good
    function dragUpload () {
      return {
        controllerAs: 'dragUpload',
        controller: function () {

        }
      };
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);
    ```

**[Back to top](#table-of-contents)**

## Filters

  - **グローバル filters**: `angular.filter()` を使ってグローバルな filter を作成し、controller / service 内でローカルな filter を作成しない

    ```javascript
    // bad
    function SomeCtrl () {
      this.startsWithLetterA = function (items) {
        return items.filter(function (item) {
          return /$a/i.test(item.name);
        });
      };
    }
    angular
      .module('app')
      .controller('SomeCtrl', SomeCtrl);

    // good
    function startsWithLetterA () {
      return function (items) {
        return items.filter(function (item) {
          return /$a/i.test(item.name);
        });
      };
    }
    angular
      .module('app')
      .filter('startsWithLetterA', startsWithLetterA);
    ```

  - テストのしやすさと再利用性を高めるため

**[Back to top](#table-of-contents)**

## Routing resolves

  - **Promises**: `$routeProvider`（または `ui-router` の `$stateProvider`）内で controller の依存を解決する

    ```javascript
    // bad
    function MainCtrl (SomeService) {
      var _this = this;
      // unresolved
      _this.something;
      // resolved asynchronously
      SomeService.doSomething().then(function (response) {
        _this.something = response;
      });
    }
    angular
      .module('app')
      .controller('MainCtrl', MainCtrl);

    // good
    function config ($routeProvider) {
      $routeProvider
      .when('/', {
        templateUrl: 'views/main.html',
        resolve: {
          // resolve here
        }
      });
    }
    angular
      .module('app')
      .config(config);
    ```

  - **Controller.resolve プロパティ**: ロジックを router にバインドせず、controller の `resolve` プロパティでロジックを関連付ける

    ```javascript
    // bad
    function MainCtrl (SomeService) {
      this.something = SomeService.something;
    }

    function config ($routeProvider) {
      $routeProvider
      .when('/', {
        templateUrl: 'views/main.html',
        controllerAs: 'main',
        controller: 'MainCtrl'
        resolve: {
          doSomething: function () {
            return SomeService.doSomething();
          }
        }
      });
    }

    // good
    function MainCtrl (SomeService) {
      this.something = SomeService.something;
    }

    MainCtrl.resolve = {
      doSomething: function () {
        return SomeService.doSomething();
      }
    };

    function config ($routeProvider) {
      $routeProvider
      .when('/', {
        templateUrl: 'views/main.html',
        controllerAs: 'main',
        controller: 'MainCtrl'
        resolve: MainCtrl.resolve
      });
    }
    ```

  - こうすることで controller と同じファイル内に resolve の依存を持たせ、router をロジックから開放する

**[Back to top](#table-of-contents)**

## Publish and subscribe events

  - **$scope**: scope 間をつなぐトリガーイベントとして `$emit` と `$broadcast` を使う

    ```javascript
    // up the $scope
    $scope.$emit('customEvent', data);

    // down the $scope
    $scope.$broadcast('customEvent', data);
    ```

  - **$rootScope**: アプリケーション全体のイベントとして `$emit` を使い、アンバインドするリスナーを忘れないようにする

    ```javascript
    // all $rootScope.$on listeners
    $rootScope.$emit('customEvent', data);
    ```

  - ヒント: `$rootScope.$on` リスナーは、`$scope.$on` リスナーと異って常に残存するため、関連する `$scope` が `$destroy` イベントを発生させたときに破棄する必要がある

    ```javascript
    // call the closure
    var unbind = $rootScope.$on('customEvent'[, callback]);
    $scope.$on('$destroy', unbind);
    ```

  - `$rootScope` リスナーが複数ある場合には、Object リテラルとループを利用して `$destroy` イベント時に自動的にアンバインドする

    ```javascript
    var rootListeners = {
      'customEvent1': $rootScope.$on('customEvent1'[, callback]),
      'customEvent2': $rootScope.$on('customEvent2'[, callback]),
      'customEvent3': $rootScope.$on('customEvent3'[, callback])
    };
    for (var unbind in rootListeners) {
      $scope.$on('$destroy', rootListeners[unbind]);
    }
    ```

**[Back to top](#table-of-contents)**

## Angular ラッパー参照

  - **$document と $window**: `$document` と `$window` を常に利用する

    ```javascript
    // bad
    function dragUpload () {
      return {
        link: function (scope, element, attrs) {
          document.addEventListener('click', function () {

          });
        }
      };
    }

    // good
    function dragUpload () {
      return {
        link: function (scope, element, attrs, $document) {
          $document.addEventListener('click', function () {

          });
        }
      };
    }
    ```

  - **$timeout と $interval**: Angular の双方向データバインドを維持するために `$timeout` と `$interval` を利用する

    ```javascript
    // bad
    function dragUpload () {
      return {
        link: function (scope, element, attrs) {
          setTimeout(function () {
            //
          }, 1000);
        }
      };
    }

    // good
    function dragUpload () {
      return {
        link: function (scope, element, attrs, $timeout) {
          $timeout(function () {
            //
          }, 1000);
        }
      };
    }
    ```

**[Back to top](#table-of-contents)**

## コメント

  - **jsDoc**: function 名、説明、パラメータ、返り値のドキュメント化に jsDoc 構文を使用する

    ```javascript
    /**
     * @name SomeService
     * @desc Main application Controller
     */
    function SomeService (SomeService) {

      /**
       * @name doSomething
       * @desc Does something awesome
       * @param {Number} x First number to do something with
       * @param {Number} y Second number to do something with
       * @returns {Number}
       */
      this.doSomething = function (x, y) {
        return x * y;
      };

    }
    angular
      .module('app')
      .service('SomeService', SomeService);
    ```

**[Back to top](#table-of-contents)**

## Minification と annotation

  - **ng-annotate**: `ng-min` が deprecated であり [ng-annotate](//github.com/olov/ng-annotate) for Gulp を利用し、`/** @ngInject */` コメントを使用して自動的に function を DI（dependency injection）させる

    ```javascript
    /**
     * @ngInject
     */
    function MainCtrl (SomeService) {
      this.doSomething = SomeService.doSomething;
    }
    angular
      .module('app')
      .controller('MainCtrl', MainCtrl);
    ```

  - `$inject` アノテーションにより以下の出力となる

    ```javascript
    /**
     * @ngInject
     */
    function MainCtrl (SomeService) {
      this.doSomething = SomeService.doSomething;
    }
    MainCtrl.$inject = ['SomeService'];
    angular
      .module('app')
      .controller('MainCtrl', MainCtrl);
    ```

**[Back to top](#table-of-contents)**

## Angular docs
API リファレンスなど、その他の情報は [Angular documentation](//docs.angularjs.org/api) を確認する。

## Contributing

Open an issue first to discuss potential changes/additions.

## License

#### (The MIT License)

Copyright (c) 2014 Todd Motto

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
