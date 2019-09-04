<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-08-19 15:29:12
 * @LastEditTime: 2019-08-19 15:34:38
 * @LastEditors: Please set LastEditors
 -->
使用pointer-events来阻止元素成为鼠标事件目标不一定意味着元素上的事件侦听器永远不会触发。如果元素后代明确指定了pointer-events属性并允许其成为鼠标事件的目标，那么指向该元素的任何事件在事件传播过程中都将通过父元素，并以适当的方式触发其上的事件侦听器。当然，位于父元素但不在后代元素上的鼠标活动都不会被父元素和后代元素捕获（鼠标活动将会穿过父元素而指向位于其下面的元素）


该属性也可用来提高滚动时的帧频。的确，当滚动时，鼠标悬停在某些元素上，则触发其上的hover效果，然而这些影响通常不被用户注意，并多半导致滚动出现问题。对body元素应用pointer-events：none，禁用了包括hover在内的鼠标事件，从而提高滚动性能。


## touch-action 
