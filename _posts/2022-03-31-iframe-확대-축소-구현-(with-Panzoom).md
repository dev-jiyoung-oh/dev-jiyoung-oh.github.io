---
layout: post
categories: 프로젝트
author: 징어군
tags: project iframe pinch-zoom panzoom
---

## iframe 확대/축소 구현 (with Panzoom)

※ 주의! 크로스 도메인(CORS)에 걸리지 않는 경우만 구현이 가능합니다...!!

### 문제해결 소스

#### HTML:
```html
<script src="https://unpkg.com/@panzoom/panzoom@4.4.4/dist/panzoom.min.js"></script>

<div id="panzoom_parent">
  <div id="panzoom">
    <div id="layer" class="full-size"></div><iframe src="해당링크" onload="iframeCssChange();" style="visibility: hidden; overflow: hidden;"></iframe>
  </div>
</div>
```
#### CSS:
```css
#layer {
  position: absolute;
  opacity: 0;
  z-index: 2;
}
```
#### JAVASCRIPT:
```javascript
// 확대/축소 설정
Panzoom(document.querySelector('#panzoom'), {
  minScale: 1,
  enableTextSelection : true,
  canvas: true, // panzoom 기능 적용한 범위 말고 다른 영역을 드래그해서도 이동시킬 수 있다.
  // contain: 'outside', // 이동 범위를 제한하는 기능인데 제대로 동작을 안해서... 일단 주석처리
});

// iframe css 변경
var iframeCssChange = function() {
  if ($('iframe').contents().find('html')) {
			var iframeInitHeight = $('iframe').height();
			var innerHtml = $('iframe').contents().find('html')[0];
			var innerWidth = innerHtml.scrollWidth;
			var innerHeight = innerHtml.scrollHeight;
			var scale = +(Math.floor(($('iframe').width() / innerWidth) + "e+2")  + "e-2"); // 소수점 둘째 자리까지
			
			$('iframe').contents().find('html').css({'transform': 'scale(' + scale + ')', 'transform-origin' : '0% 0%', 'width' : innerWidth, 'height' : '0'});
			
      // iframe 내부 영역의 크기와 iframe 크기를 비교해서 내부의 크기를 iframe 크기에 딱 맞게 정적으로 확대/
			if (innerHeight * scale > iframeInitHeight) {
				$('iframe').height(innerHeight * scale);
				$('#layer').height(innerHeight * scale);
			}
		
			// iframe 내부 링크 처리
			$('iframe').contents().find('[href]').each(function(){
				var href = $(this).attr('href');
				$(this).attr({
					'href': 'javascript:void(0);',
					'data-href': href
				});
			}).on("click",function(){
        // 클릭 시 실행
				var href = $(this).data("href");
        // ...
			});
			
			// iframe 내부로 클릭 이벤트 전달
			$(document).on("click", "#layer", function(event){
				var iframe = $('iframe').get(0);
				var iframeDoc = (iframe.contentDocument) ? iframe.contentDocument : iframe.contentWindow.document;
				
				// Find click position (coordinates)
				var x = event.offsetX;
				var y = event.offsetY;
				
				// Trigger click inside iframe
				var link = iframeDoc.elementFromPoint(x, y);
				var newEvent = iframeDoc.createEvent('HTMLEvents');
				newEvent.initEvent('click', true, true);
				link.dispatchEvent(newEvent);
			});
		}
    
	  $('iframe').css('visibility', '');
};
```

### 아직 해결하지 못한 사항!!!
1. 이동 범위 제한
현재는 화면에서 iframe이 어디로든 움직일 수 있다. iframe이 안 보이는 곳으로도 이동 가능...
2. 크로스 도메인 문제...
iframe 내부 요소의 너비, 높이를 알아내서 iframe 크기에 맞게 scale을 조절시키는게 이 확대/축소 구현의 관건인데
크로스 도메인 정책때문에 현재 도메인과 iframe 내부에 호출하는 도메인이 다를 경우 처리가 불가능하다...ㅠㅠ


### 기능 구현까지의 서사

#### iframe 확대/축소 기능의 필요성
하이브리드 모바일 앱을 구축하면서 공지사항을 구현해야 하는데 상세 내용에 웹 페이지를 보여줘야 하는 거라서 iframe을 이용하게 되었다.

상세 내용을 스크롤 없이 iframe 크기에 맞춰 모두 보여주려고 하는데 (iframe크기와 내부요소의 크기 비율을 계산해서 scale을 구해서 iframe 내부 요소에 적용)

상세 내용이 웹을 기준으로 작성되어 있다보니, 표나 사진이 있을 경우 글자 크기가 너무 작아서 확대/축소가 필수적이었다..

#### iframe 내부 확대/축소X. iframe 자체를 확대/축소
iframe의 내부 요소 자체를 확대/축소하는 방법은 아직 찾지 못했다..

**그래서 iframe 내부 요소를 iframe 크기에 딱 맞춰서 내부 html에 scale을 적용하고 iframe 자체를 확대/축소하면 되지 않을까 하고 방법을 찾아봤다.**

#### Panzoom과의 만남..그리고 문제점
iframe 확대/축소를 구글링하니 scale을 이용해서 정적으로 처리하는 방법만 주구장창 나왔는데

그러다가 한줄기 빛과 같은 Panzoom을 만났다.

[Panzoom Github 주소](https://github.com/timmywil/panzoom) [Panzoom Demo](https://timmywil.com/panzoom/demo/)

Panzoom 라이브러리는 scale등과 같은 css요소를 이용해서 확대/축소를 할 수 있도록 도와준다고 한다.

Panzoom 깃허브 설명에

```
Panzoom uses CSS transforms to take advantage of hardware/GPU acceleration in the browser, which means the element can be anything: an image, a video, an iframe, a canvas, text, WHATEVER.
```

라고 적혀있는데... 문제는.....

#### 문제 part 1
**다른 애들은 다 되는데 iframe만 확대/축소가 안되잖아...!!!!!!!!!!!!!**

그래서 원인이 뭘까 생각하다가


#### 문제해결 part 1
iframe의 스크롤 이벤트랑 Panzoom 라이브러리의 핀치 이벤트가 서로 충돌해서
Panzoom의 핀치 줌 기능이 iframe에서는 제대로 동작하지 않는거라는 생각이 들어서 

아예 아이프레임의 스크롤 기능이 작동하지 않게 가려버리면 되지 않을까 생각해서 아래와 같이 구현했다.

```html
<div id="wrap">
  <div id="layer"></div>
  <iframe src="링크"/>
</div>
```
이렇게 하고 #wrap에 panzoom 기능을 적용하니까 성공적으로 iframe 확대 축소가 가능했다!!

예~~~ 드디어 성공!!! 하고 기뻐하려는 찰나....

#### 문제 part 2
**상세 웹페이지에서 링크 이동을 하는 경우가 있다는 것을 알게 되었다...**

그리고 중요한 건 iframe 내부의 링크를 클릭했을 때 

**iframe 내부에서 이동하지 않고 특정 함수를 호출해서 외부 앱(예: 삼성인터넷)에서 해당 페이지가 열리게끔 처리해야 한다는 것이었다.**

이 말은 즉, 아이프레임 내부의 href 요소를 지우고 해당 태그에 내가 원하는 함수를 click 이벤트에 걸어야 한다는 것!

#### 문제해결 part 2
오랫동안 이 기능을 어떻게 구현하나 걱정하다가 오늘 딱!!! 구글링하다가 해결방안을 찾았다.

##### 구세주님 좌표 [https://stackoverflow.com/questions/15248970/iframe-prevents-iscroll-scrolling-on-mobile-safari](https://stackoverflow.com/questions/15248970/iframe-prevents-iscroll-scrolling-on-mobile-safari)

구현방법은 

iframe 위에 레이어 div를 배치하고,
해당 iframe 요소를 탭/클릭하려는 경우
레이어 div에서 감지한 x, y 좌표를 저장하고
iframe 콘텐츠 내의 해당 좌표에 대해 클릭 이벤트를 발생시키는 방식!

결과물은 [위](#문제해결-소스)와 같다~

아직 해결해야 하는 것들이 많이 있지만...이렇게라도 구현한 것에...나름 만족....!

