# AngularJS スタイルガイド

*チーム開発のための AngularJS スタイルガイド by [@toddmotto](//twitter.com/toddmotto)*

チームで AngularJS アプリケーションを開発するための標準的なスタイルガイドとして、Angular アプリケーションについてのこれまでの[記事](http:////toddmotto.com)、[講演](https://speakerdeck.com/toddmotto)、そして構築してきた経験を基にして、コンセプト、シンタックス、規約についてまとめている。

#### コミュニティ
[John Papa](//twitter.com/John_Papa) と私は Angular スタイルのパターンについて話し合い、それによってこのガイドはよりすばらしいものとなっている。それぞれのスタイルガイドをリリースしているので、考えを比較するためにも [John のスタイルガイド](//github.com/johnpapa/angularjs-styleguide)もぜひ確認してみてほしい。

> See the [original article](http://toddmotto.com/opinionated-angular-js-styleguide-for-teams) that sparked this off

## Table of Contents

  1. [Modules](#modules)
  1. [Controllers](#controllers)
  1. [Services and Factory](#services-and-factory)
  1. [Directives](#directives)
  1. [Filters](#filters)
  1. [Routing resolves](#routing-resolves)
  1. [Publish and subscribe events](#publish-and-subscribe-events)
  1. [Performance](#performance)
  1. [Angular wrapper references](#angular-wrapper-references)
  1. [Comment standards](#comment-standards)
  1. [Minification and annotation](#minification-and-annotation)

## Modules

  - **定義**: 変数を使わずに setter / getter で module を定義

    ```javascript
    // avoid
    var app = angular.module('app', []);
    app.controller();
    app.factory();

    // recommended
    angular
      .module('app', [])
      .controller()
      .factory();
    ```

  - Note: `angular.module('app', []);` を setter、`angular.module('app');` を getter として使う。setter で module を定義し、他のインスタンスからは getter でその module を取得して利用する。

  - **メソッド**: コールバックとして記述せず、function を定義してメソッドに渡す

    ```javascript
    // avoid
    angular
      .module('app', [])
      .controller('MainCtrl', function MainCtrl () {

      })
      .service('SomeService', function SomeService () {

      });

    // recommended
    function MainCtrl () {

    }
    function SomeService () {

    }
    angular
      .module('app', [])
      .controller('MainCtrl', MainCtrl)
      .service('SomeService', SomeService);
    ```

  - コードのネストが深くなることを抑え、可読性を高められる
  
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

  - **controllerAs**: Controller はクラスであるため、常に `controllerAs` を利用する

    ```html
    <!-- avoid -->
    <div ng-controller="MainCtrl">
      {{ someObject }}
    </div>

    <!-- recommended -->
    <div ng-controller="MainCtrl as main">
      {{ main.someObject }}
    </div>
    ```

  - DOM で controller ごとに変数を定義し、`$parent` の利用を避ける

  - `controllerAs` では controller 内で `$scope` にバインドされる `this` を利用する

    ```javascript
    // avoid
    function MainCtrl ($scope) {
      $scope.someObject = {};
      $scope.doSomething = function () {

      };
    }

    // recommended
    function MainCtrl () {
      this.someObject = {};
      this.doSomething = function () {

      };
    }
    ```

  - `$emit`、`$broadcast`、`$on` や `$watch` で必要とならない限り、`controllerAs` では `$scope` を利用しない

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

  - **controllerAs 'vm'**: controller の `this` コンテキストを、`ViewModel` を意味する `vm` として保持する

    ```javascript
    // avoid
    function MainCtrl () {
      this.doSomething = function () {

      };
    }

    // recommended
    function MainCtrl (SomeService) {
      var vm = this;
      vm.doSomething = SomeService.doSomething;
    }
    ```

    *Why?* : Function コンテキストが `this` の値を変えてしまうことによる `.bind()` の利用とスコープの問題を回避するため

  - **プレゼンテーションロジックのみ (MVVM)**: controller 内ではプレゼンテーションロジックのみとし、ビジネスロジックは service に委譲する

    ```javascript
    // avoid
    function MainCtrl () {
      
      var vm = this;

      $http
        .get('/users')
        .success(function (response) {
          vm.users = response;
        });

      vm.removeUser = function (user, index) {
        $http
          .delete('/user/' + user.id)
          .then(function (response) {
            vm.users.splice(index, 1);
          });
      };

    }

    // recommended
    function MainCtrl (UserService) {

      var vm = this;

      UserService
        .getUsers()
        .then(function (response) {
          vm.users = response;
        });

      vm.removeUser = function (user, index) {
        UserService
          .removeUser(user)
          .then(function (response) {
            vm.users.splice(index, 1);
          });
      };

    }
    ```

    *Why?* : controller では service からモデルのデータを取得するようにしてビジネスロジックを避け、ViewModel としてモデル・ビュー間のデータフローを制御させる。controller 内のビジネスロジックは service のテストを不可能にしてしまう。

**[Back to top](#table-of-contents)**

## Services and Factory

  - すべての Angular Services はシングルトンで、`.service()` と `.factory()` はオブジェクトの生成され方が異なる

  **Services**: `constructor` function として `new` で生成し、パブリックなメソッドと変数に `this` を使う

    ```javascript
    function SomeService () {
      this.someMethod = function () {

      };
    }
    angular
      .module('app')
      .service('SomeService', SomeService);
    ```

  **Factory**: ビジネスロジックやプロバイダモジュールで、オブジェクトやクロージャを返す

  - 常にホストオブジェクトを返す

    ```javascript
    function AnotherService () {
      var AnotherService = {};
      AnotherService.someValue = '';
      AnotherService.someMethod = function () {

      };
      return AnotherService;
    }
    angular
      .module('app')
      .factory('AnotherService', AnotherService);
    ```

    *Why?* : "Revealing Module Pattern" では primitive な値は更新されない

**[Back to top](#table-of-contents)**

## Directives

  - **restrict**: 独自 directive には `custom element` と `custom attribute` のみ利用する（`{ restrict: 'EA' }`）

    ```html
    <!-- avoid -->

    <!-- directive: my-directive -->
    <div class="my-directive"></div>

    <!-- recommended -->

    <my-directive></my-directive>
    <div my-directive></div>
    ```

  - コメントとクラス名での宣言は混乱しやすいため使うべきでない。コメントでの宣言は古いバージョンの IE では動作せず、属性での宣言が古いブラウザをカバーするのにもっとも安全である。

  - **template**: テンプレートをすっきりさせるために `Array.join('')` を利用する

    ```javascript
    // avoid
    function someDirective () {
      return {
        template: '<div class="some-directive">' +
          '<h1>My directive</h1>' +
        '</div>'
      };
    }

    // recommended
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

    *Why?* : 適切なインデントでコードの可読性を高められ、不適切に `+` を使ってしまうことによるエラーを避けられる

  - **DOM 操作**: directive 内のみとし、controller / service では DOM を操作しない

    ```javascript
    // avoid
    function UploadCtrl () {
      $('.dragzone').on('dragend', function () {
        // handle drop functionality
      });
    }
    angular
      .module('app')
      .controller('UploadCtrl', UploadCtrl);

    // recommended
    function dragUpload () {
      return {
        restrict: 'EA',
        link: function ($scope, $element, $attrs) {
          $element.on('dragend', function () {
            // handle drop functionality
          });
        }
      };
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);
    ```

  - **命名規約**: 将来的に標準 directive と名前が衝突する可能性があるため、`ng-*` を独自 directive に使わない

    ```javascript
    // avoid
    // <div ng-upload></div>
    function ngUpload () {
      return {};
    }
    angular
      .module('app')
      .directive('ngUpload', ngUpload);

    // recommended
    // <div drag-upload></div>
    function dragUpload () {
      return {};
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);
    ```

  - directive と filter は先頭文字を小文字で命名する。これは、Angular が `camelCase` をハイフンつなぎとする命名規約によるもので、`dragUpload` が要素で使われた場合は `<div drag-upload></div>` となる。

  - **controllerAs**: directive でも `controllerAs` を使う

    ```javascript
    // avoid
    function dragUpload () {
      return {
        controller: function ($scope) {

        }
      };
    }
    angular
      .module('app')
      .directive('dragUpload', dragUpload);

    // recommended
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

  - **グローバル filter**: angular.filter() を使ってグローバルな filter を作成し、controller / service 内でローカルな filter を使わない

    ```javascript
    // avoid
    function SomeCtrl () {
      this.startsWithLetterA = function (items) {
        return items.filter(function (item) {
          return /^a/i.test(item.name);
        });
      };
    }
    angular
      .module('app')
      .controller('SomeCtrl', SomeCtrl);

    // recommended
    function startsWithLetterA () {
      return function (items) {
        return items.filter(function (item) {
          return /^a/i.test(item.name);
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
    // avoid
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

    // recommended
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
    // avoid
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

    // recommended
    function MainCtrl (SomeService) {
      this.something = SomeService.something;
    }

    MainCtrl.resolve = {
      doSomething: function (SomeService) {
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

  - **$scope**: scope 間をつなぐイベントトリガーとして `$emit` と `$broadcast` を使う

    ```javascript
    // up the $scope
    $scope.$emit('customEvent', data);

    // down the $scope
    $scope.$broadcast('customEvent', data);
    ```

  - **$rootScope**: アプリケーション全体のイベントとして `$emit` を使い、忘れずにリスナーをアンバインドする

    ```javascript
    // all $rootScope.$on listeners
    $rootScope.$emit('customEvent', data);
    ```

  - ヒント: `$rootScope.$on` リスナーは、`$scope.$on` リスナーと異なり常に残存するため、関連する `$scope` が `$destroy` イベントを発生させたときに破棄する必要がある

    ```javascript
    // call the closure
    var unbind = $rootScope.$on('customEvent'[, callback]);
    $scope.$on('$destroy', unbind);
    ```

  - `$rootScope` リスナーが複数ある場合は、Object リテラルでループして `$destroy` イベント時に自動的にアンバインドさせる

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

## Performance

  - **ワンタイムバインド**: Angular の新しいバージョン（v1.3.0-beta.10+）では、ワンタイムバインドのシンタックス `{{ ::value }}` を利用する

    ```html
    // avoid
    <h1>{{ vm.title }}</h1>

    // recommended
    <h1>{{ ::vm.title }}</h1>
    ```
    
    *Why?* : `undefined` の変数が解決されたときに `$$watchers` から取り除き、ダーティチェックでのパフォーマンスを改善する
    
  - **$scope.$digest を検討**: `$scope.$apply` でなく `$scope.$digest` を使い、子スコープのみを更新する

    ```javascript
    $scope.$digest();
    ```
    
    *Why?* : `$scope.$apply` は `$rootScope.$digest` を呼び出すため、アプリケーション全体の `$$watchers` をダーティチェックするが、`$scope.$digest` は `$scope` のスコープと子スコープを更新する

**[Back to top](#table-of-contents)**

## Angular wrapper references

  - **$document と $window**: `$document` と `$window` を常に利用する

    ```javascript
    // avoid
    function dragUpload () {
      return {
        link: function ($scope, $element, $attrs) {
          document.addEventListener('click', function () {

          });
        }
      };
    }

    // recommended
    function dragUpload () {
      return {
        link: function ($scope, $element, $attrs, $document) {
          $document.addEventListener('click', function () {

          });
        }
      };
    }
    ```

  - **$timeout と $interval**: Angular の双方向データバインドが最新の状態を維持するよう `$timeout` と `$interval` を利用する

    ```javascript
    // avoid
    function dragUpload () {
      return {
        link: function ($scope, $element, $attrs) {
          setTimeout(function () {
            //
          }, 1000);
        }
      };
    }

    // recommended
    function dragUpload ($timeout) {
      return {
        link: function ($scope, $element, $attrs) {
          $timeout(function () {
            //
          }, 1000);
        }
      };
    }
    ```

**[Back to top](#table-of-contents)**

## Comment standards

  - **jsDoc**: jsDoc で function 名、説明、パラメータ、返り値をドキュメント化する

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

## Minification and annotation

  - **ng-annotate**: `ng-min` は deprecated なので、[ng-annotate](//github.com/olov/ng-annotate) for Gulp を利用し、`/** @ngInject */` で function にコメントして自動的に DI (dependency injection) させる

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

  - 以下のような `$inject` アノテーションを含む出力となる

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
その他、API リファレンスなどの情報は、[Angular documentation](//docs.angularjs.org/api) を確認する。

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
