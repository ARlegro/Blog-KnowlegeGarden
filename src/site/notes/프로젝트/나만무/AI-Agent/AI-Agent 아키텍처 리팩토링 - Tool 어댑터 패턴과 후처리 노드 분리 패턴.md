---
{"dg-publish":true,"permalink":"/프로젝트/나만무/AI-Agent/AI-Agent 아키텍처 리팩토링 - Tool 어댑터 패턴과 후처리 노드 분리 패턴/","noteIcon":"","created":"2025-12-06T12:52:49.591+09:00","updated":"2025-12-10T10:36:58.294+09:00"}
---


## 1.  최종 아키텍쳐 Preview 
<style> .container {font-family: sans-serif; text-align: center;} .button-wrapper button {z-index: 1;height: 40px; width: 100px; margin: 10px;padding: 5px;} .excalidraw .App-menu_top .buttonList { display: flex;} .excalidraw-wrapper { height: 800px; margin: 50px; position: relative;} :root[dir="ltr"] .excalidraw .layer-ui__wrapper .zen-mode-transition.App-menu_bottom--transition-left {transform: none;} </style><script src="https://cdn.jsdelivr.net/npm/react@17/umd/react.production.min.js"></script><script src="https://cdn.jsdelivr.net/npm/react-dom@17/umd/react-dom.production.min.js"></script><script type="text/javascript" src="https://cdn.jsdelivr.net/npm/@excalidraw/excalidraw@0/dist/excalidraw.production.min.js"></script><div id="Drawing_2025-12-05_0015.57.excalidraw.md1"></div><script>(function(){const InitialData={"type":"excalidraw","version":2,"source":"https://github.com/zsviczian/obsidian-excalidraw-plugin/releases/tag/2.18.0","elements":[{"id":"X3z3-gUWVyLHWUOT5uDmH","type":"rectangle","x":-152.0666097005207,"y":-56.91040039062504,"width":212,"height":125.60000610351562,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a0","roundness":{"type":3},"seed":682187386,"version":115,"versionNonce":2094132685,"isDeleted":false,"boundElements":[{"type":"text","id":"phanobym"},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow"},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow"},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow"},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow"},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow"},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"phanobym","type":"text","x":-120.64457448323554,"y":-26.310397338867226,"width":149.1559295654297,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a1","roundness":null,"seed":1012047078,"version":215,"versionNonce":109998159,"isDeleted":false,"boundElements":[],"updated":1765293195538,"link":null,"locked":false,"text":"Agent_node\n(두뇌)","rawText":"Agent_node\n(두뇌)","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"X3z3-gUWVyLHWUOT5uDmH","originalText":"Agent_node\n(두뇌)","autoResize":true,"lineHeight":1.15},{"id":"OH2497AIboS6cZZRICftD","type":"rectangle","x":-137.66642252604169,"y":138.55628458658856,"width":191.99995930989584,"height":93.066660563151,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a2","roundness":{"type":3},"seed":1759268454,"version":138,"versionNonce":1045974157,"isDeleted":false,"boundElements":[{"type":"text","id":"p5d56VBH"},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow"},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow"},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow"},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow"}],"updated":1765423715410,"link":null,"locked":false},{"id":"p5d56VBH","type":"text","x":-102.20242309570314,"y":168.98961486816407,"width":121.07196044921875,"height":32.199999999999996,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a3","roundness":null,"seed":1077162490,"version":174,"versionNonce":1729728111,"isDeleted":false,"boundElements":[],"updated":1765293195538,"link":null,"locked":false,"text":"Tool Layer","rawText":"Tool Layer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"OH2497AIboS6cZZRICftD","originalText":"Tool Layer","autoResize":true,"lineHeight":1.15},{"id":"YmNG4ENWSaXq9ugMgHRXY","type":"ellipse","x":-134.46634928385419,"y":292.9561564127607,"width":249.86661783854169,"height":163,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a6","roundness":{"type":2},"seed":1924747302,"version":281,"versionNonce":1800229059,"isDeleted":false,"boundElements":[{"type":"text","id":"FFfqsG6c"},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow"},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow"},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow"},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow"},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow"}],"updated":1765423733134,"link":null,"locked":false},{"id":"FFfqsG6c","type":"text","x":-89.04819575157511,"y":342.1269537460571,"width":159.34793090820312,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a7","roundness":null,"seed":1037160294,"version":324,"versionNonce":353514189,"isDeleted":false,"boundElements":[],"updated":1765423715409,"link":null,"locked":false,"text":"route_after_\ntools","rawText":"route_after_tools","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"YmNG4ENWSaXq9ugMgHRXY","originalText":"route_after_tools","autoResize":true,"lineHeight":1.15},{"id":"JtVq8TF3VQFwMa_Am6Wgi","type":"rectangle","x":-296.599873860677,"y":596.4228719075521,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a8","roundness":{"type":3},"seed":1361230246,"version":135,"versionNonce":931090669,"isDeleted":false,"boundElements":[{"type":"text","id":"6PxyvEs3"},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow"},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow"}],"updated":1765423733134,"link":null,"locked":false},{"id":"6PxyvEs3","type":"text","x":-288.6878102620442,"y":633.4228841145833,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a9","roundness":null,"seed":849849766,"version":197,"versionNonce":174546701,"isDeleted":false,"boundElements":[],"updated":1765423733574,"link":null,"locked":false,"text":"handle Node\n1","rawText":"handle Node 1","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"JtVq8TF3VQFwMa_Am6Wgi","originalText":"handle Node 1","autoResize":true,"lineHeight":1.15},{"id":"ooSxET1T42g_5rrv2Buu6","type":"rectangle","x":-101.71778517502997,"y":601.7255687224559,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aA","roundness":{"type":3},"seed":536348410,"version":165,"versionNonce":679551651,"isDeleted":false,"boundElements":[{"type":"text","id":"USqQBn57"},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow"},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"USqQBn57","type":"text","x":-93.80572157639716,"y":638.7255809294871,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aB","roundness":null,"seed":1343297466,"version":225,"versionNonce":755153891,"isDeleted":false,"boundElements":[],"updated":1765423716359,"link":null,"locked":false,"text":"handle Node\n2","rawText":"handle Node 2","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"ooSxET1T42g_5rrv2Buu6","originalText":"handle Node 2","autoResize":true,"lineHeight":1.15},{"id":"6IA8bLzWF3smACML4jSQf","type":"rectangle","x":113.19502532176489,"y":604.2690867888622,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aC","roundness":{"type":3},"seed":1110571750,"version":165,"versionNonce":2102176813,"isDeleted":false,"boundElements":[{"type":"text","id":"ctSJUZff"},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow"},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow"},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"ctSJUZff","type":"text","x":121.1070889203977,"y":641.2690989958934,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aD","roundness":null,"seed":624799270,"version":224,"versionNonce":967264131,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"text":"handle Node\n3","rawText":"handle Node 3","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"6IA8bLzWF3smACML4jSQf","originalText":"handle Node 3","autoResize":true,"lineHeight":1.15},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow","x":-44.96001833907274,"y":79.6896057128906,"width":0.3780108839171419,"height":47.866678873697964,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aE","roundness":{"type":2},"seed":1592951034,"version":187,"versionNonce":1282525165,"isDeleted":false,"boundElements":[],"updated":1765423716367,"link":null,"locked":false,"points":[[0,0],[-0.3780108839171419,47.866678873697964]],"lastCommittedPoint":null,"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.5079317665142968,0.5079317665142967]},"endBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.4785926893582332,0.47859268935823324]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow","x":-35.97674949446286,"y":242.6229451497396,"width":7.71713564728401,"height":40.21517757733281,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aF","roundness":{"type":2},"seed":2066691110,"version":180,"versionNonce":1050717005,"isDeleted":false,"boundElements":[],"updated":1765423733134,"link":null,"locked":false,"points":[[0,0],[7.71713564728401,40.21517757733281]],"lastCommittedPoint":[5.333333333333314,133.3333333333333],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.47450320286991415,0.5254967971300856]},"endBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.49542835966062526,0.5001]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow","x":-93.81644360152464,"y":447.02926860582744,"width":110.27659416758468,"height":138.39360330172468,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aG","roundness":{"type":2},"seed":1846062010,"version":238,"versionNonce":711654189,"isDeleted":false,"boundElements":[],"updated":1765423733137,"link":null,"locked":false,"points":[[0,0],[-110.27659416758468,138.39360330172468]],"lastCommittedPoint":[-238.93326822916663,164.26664225260424],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.39407252956333966,0.5001]},"endBinding":{"elementId":"JtVq8TF3VQFwMa_Am6Wgi","mode":"orbit","fixedPoint":[0.30531257047929755,0.3053125704792968]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow","x":-12.347064349429665,"y":466.936333745727,"width":2.236567317393508,"height":123.78923497672895,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aH","roundness":{"type":2},"seed":1943452858,"version":165,"versionNonce":150862861,"isDeleted":false,"boundElements":[],"updated":1765423733135,"link":null,"locked":false,"points":[[0,0],[2.236567317393508,123.78923497672895]],"lastCommittedPoint":[28.79996744791663,167.46663411458326],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.482051947950208,0.5001]},"endBinding":{"elementId":"ooSxET1T42g_5rrv2Buu6","mode":"orbit","fixedPoint":[0.5664038097391358,0.43359619026086504]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow","x":64.38300120304834,"y":452.08551034168437,"width":142.24318647083456,"height":141.18357644717793,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aI","roundness":{"type":2},"seed":1506914790,"version":169,"versionNonce":1029501549,"isDeleted":false,"boundElements":[],"updated":1765423733135,"link":null,"locked":false,"points":[[0,0],[142.24318647083456,141.18357644717793]],"lastCommittedPoint":[313.59993489583337,177.06669108072924],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.5001,0.5263112548480741]},"endBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[0.8039593463345159,0.19604065366548287]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow","x":-305.61116683390895,"y":612.810835820468,"width":165.00685102272652,"height":611.9994610187518,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aJ","roundness":{"type":2},"seed":103747962,"version":391,"versionNonce":963084995,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[-22.46229388933824,-540.8442968492684],[142.54455713338828,-611.9994610187518]],"lastCommittedPoint":[221.53846153846155,-733.5384427584133],"startBinding":{"elementId":"JtVq8TF3VQFwMa_Am6Wgi","mode":"orbit","fixedPoint":[-0.05513419038195911,0.11366915387463353]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.24411731487226143,0.24411731487226132]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow","x":69.27028907568963,"y":612.5183261619767,"width":95.37441175955671,"height":599.4476452992812,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aK","roundness":{"type":2},"seed":333631462,"version":387,"versionNonce":349781741,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[95.37441175955671,-426.0902800293187],[1.663101223789667,-599.4476452992812]],"lastCommittedPoint":[19.692195012019283,-695.3846153846155],"startBinding":{"elementId":"ooSxET1T42g_5rrv2Buu6","mode":"orbit","fixedPoint":[0.8893468411323211,0.889346841132321]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.9041445704193891,0.09585542958061087]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow","x":272.4719744028724,"y":600.712209764589,"width":201.53858410339308,"height":593.5726680637304,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aM","roundness":{"type":2},"seed":1858775674,"version":351,"versionNonce":797979235,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[-76.64826214935698,-493.13544756121803],[-201.53858410339308,-593.5726680637304]],"lastCommittedPoint":[-253.5385366586538,-735.999990609976],"startBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[1.0401707832974996,0.5001]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8135583943715337,0.18644160562846634]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"1sQyfb53KVknwBxb3567b","type":"rectangle","x":290.92003498818116,"y":181.50138495464492,"width":199.46663411458326,"height":121.60001627604163,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aN","roundness":{"type":3},"seed":1164082598,"version":198,"versionNonce":730484419,"isDeleted":false,"boundElements":[{"type":"text","id":"SN7dtwUA"},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow"}],"updated":1765423715410,"link":null,"locked":false},{"id":"SN7dtwUA","type":"text","x":347.01537444537513,"y":210.10139309266575,"width":87.27595520019531,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aO","roundness":null,"seed":1106535846,"version":257,"versionNonce":528199139,"isDeleted":false,"boundElements":[],"updated":1765423715409,"link":null,"locked":false,"text":"Service\nLayer","rawText":"Service\nLayer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"1sQyfb53KVknwBxb3567b","originalText":"Service\nLayer","autoResize":true,"lineHeight":1.15},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow","x":65.33353678385416,"y":199.03019314191843,"width":214.586498204327,"height":25.32408157876418,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aP","roundness":{"type":2},"seed":1239326758,"version":101,"versionNonce":1314524899,"isDeleted":false,"boundElements":[],"updated":1765423715420,"link":null,"locked":false,"points":[[0,0],[214.586498204327,25.32408157876418]],"lastCommittedPoint":[197.33333333333331,2.1333414713541288],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.5186504115874699,0.51865041158747]},"endBinding":{"elementId":"1sQyfb53KVknwBxb3567b","mode":"orbit","fixedPoint":[0.4502438138781704,0.45024381387817053]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"5QKjK6LP1M3apzPJ-pNek","type":"diamond","x":316.6641156841124,"y":349.9826977085189,"width":191.68035831974692,"height":188.53801963434958,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aQ","roundness":{"type":2},"seed":1390001359,"version":132,"versionNonce":389603117,"isDeleted":false,"boundElements":[{"type":"text","id":"Nq2NIrP5"},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow"},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow"},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow"},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow"}],"updated":1765423761103,"link":null,"locked":false},{"id":"Nq2NIrP5","type":"text","x":380.8042293729359,"y":428.0172026171063,"width":63.55995178222656,"height":32.199999999999996,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aR","roundness":null,"seed":506086913,"version":173,"versionNonce":1316561795,"isDeleted":false,"boundElements":[],"updated":1765423755229,"link":null,"locked":false,"text":"State","rawText":"State","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"5QKjK6LP1M3apzPJ-pNek","originalText":"State","autoResize":true,"lineHeight":1.15},{"id":"YQ5rNIULGbw2vQfz3Du0P","type":"diamond","x":285.0336028137207,"y":-29.54062398754101,"width":209.3141097151048,"height":160.2693317148178,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aS","roundness":{"type":2},"seed":1182592833,"version":328,"versionNonce":1161884867,"isDeleted":false,"boundElements":[{"type":"text","id":"SSHfSr2H"},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow"}],"updated":1765423751442,"link":null,"locked":false},{"id":"SSHfSr2H","type":"text","x":346.01415355792653,"y":18.326708941163446,"width":87.69595336914062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aT","roundness":null,"seed":712990497,"version":384,"versionNonce":594240611,"isDeleted":false,"boundElements":[],"updated":1765423751442,"link":null,"locked":false,"text":"Check\nPointer","rawText":"Check\nPointer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"YQ5rNIULGbw2vQfz3Du0P","originalText":"Check\nPointer","autoResize":true,"lineHeight":1.15},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow","x":282.3718527171511,"y":611.5231455969616,"width":83.87264672259482,"height":103.0157766932067,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aW","roundness":{"type":2},"seed":2075619841,"version":45,"versionNonce":76444045,"isDeleted":false,"boundElements":[],"updated":1765423761103,"link":null,"locked":false,"points":[[0,0],[83.87264672259482,-103.0157766932067]],"startBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[0.6274054152117697,0.6274054152117706]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.5001,0.539323457812219]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow","x":125.94297844853571,"y":382.0371151229574,"width":188.18520072945142,"height":53.118621621951036,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aX","roundness":{"type":2},"seed":186865359,"version":141,"versionNonce":456112931,"isDeleted":false,"boundElements":[],"updated":1765423755229,"link":null,"locked":false,"points":[[0,0],[188.18520072945142,53.118621621951036]],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.9294429416153576,0.5001]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.15346942903874886,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow","x":65.20922904997465,"y":212.03847310280614,"width":249.56855020162558,"height":221.90322301622874,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aY","roundness":{"type":2},"seed":1049835279,"version":322,"versionNonce":866712845,"isDeleted":false,"boundElements":[],"updated":1765423758961,"link":null,"locked":false,"points":[[0,0],[249.56855020162558,221.90322301622874]],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.7551440573671417,0.24485594263285798]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.05076345491501994,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow","x":70.84447803383041,"y":40.83692118305623,"width":245.86966452222515,"height":390.0743360198682,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aZ","roundness":{"type":2},"seed":1568001935,"version":368,"versionNonce":812801731,"isDeleted":false,"boundElements":[],"updated":1765423755230,"link":null,"locked":false,"points":[[0,0],[245.86966452222515,390.0743360198682]],"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8246251504386163,0.1753748495613837]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.044097163342550524,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow","x":70.93339029947929,"y":-18.43852436629765,"width":211.5780104038157,"height":61.96825371528639,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aa","roundness":{"type":2},"seed":1880908577,"version":210,"versionNonce":501669891,"isDeleted":false,"boundElements":[],"updated":1765423751443,"link":null,"locked":false,"points":[[0,0],[211.5780104038157,61.96825371528639]],"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8109894317567161,0.18901056824328386]},"endBinding":{"elementId":"YQ5rNIULGbw2vQfz3Du0P","mode":"orbit","fixedPoint":[0.10278159774802884,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false}],"appState":{"theme":"light","viewBackgroundColor":"#ffffff","currentItemStrokeColor":"#1e1e1e","currentItemBackgroundColor":"#eebefa","currentItemFillStyle":"solid","currentItemStrokeWidth":2,"currentItemStrokeStyle":"dotted","currentItemRoughness":1,"currentItemOpacity":100,"currentItemFontFamily":6,"currentItemFontSize":28,"currentItemTextAlign":"left","currentItemStartArrowhead":"arrow","currentItemEndArrowhead":"arrow","currentItemArrowType":"round","currentItemFrameRole":null,"scrollX":971.788479447701,"scrollY":224.94900663804094,"zoom":{"value":1},"currentItemRoundness":"round","gridSize":20,"gridStep":5,"gridModeEnabled":false,"gridColor":{"Bold":"rgba(217, 217, 217, 0.5)","Regular":"rgba(230, 230, 230, 0.5)"},"currentStrokeOptions":null,"frameRendering":{"enabled":true,"clip":true,"name":true,"outline":true,"markerName":true,"markerEnabled":true},"objectsSnapModeEnabled":false,"activeTool":{"type":"selection","customType":null,"locked":false,"fromSelection":false,"lastActiveTool":null},"disableContextMenu":false},"files":{}};InitialData.scrollToContent=true;App=()=>{const e=React.useRef(null),t=React.useRef(null),[n,i]=React.useState({width:void 0,height:void 0});return React.useEffect(()=>{i({width:t.current.getBoundingClientRect().width,height:t.current.getBoundingClientRect().height});const e=()=>{i({width:t.current.getBoundingClientRect().width,height:t.current.getBoundingClientRect().height})};return window.addEventListener("resize",e),()=>window.removeEventListener("resize",e)},[t]),React.createElement(React.Fragment,null,React.createElement("div",{className:"excalidraw-wrapper",ref:t},React.createElement(ExcalidrawLib.Excalidraw,{ref:e,width:n.width,height:n.height,initialData:InitialData,viewModeEnabled:!0,zenModeEnabled:!0,gridModeEnabled:!1})))},excalidrawWrapper=document.getElementById("Drawing_2025-12-05_0015.57.excalidraw.md1");ReactDOM.render(React.createElement(App),excalidrawWrapper);})();</script>

---
## 2.  배경 및 문제 인식 

초기 프로젝트는 기획 지연과 급박한 AI Agent 기능 도입으로 인해 코드 품질보다는 빠른 개발에 몰두하게됐다. 그 결과, 초반에는 “작동하는 것”에만 집중할 수밖에 없었고 구조적 완성도는 자연스럽게 뒤로 밀렸다.

AI Agent를 만들어보는 것은 처음이라 어떤 계층 어떤 패턴이 적절한지에 대한 기준이 부족했다. 공식 문서([LangChain, LangGraph](https://www.langchain.com/))에서도 아키텍처를 어떻게 설계하거나 계층화하라는 가이드를 보지 못해 시행착오가 많았다.

가장 맘에 안 들었던 코드는 "툴(Tool) 구조의 일관성 부족이었다.
Tool마다 반환 형식이 다르고, 처리 과정 또한 제각각이어서 다음과 같은 문제가 반복해서 발생했다.
기존 툴의 문제
```python

@tools 
def ~~ 
# ToolResult형식으로 반환하는 예시 
```
이렇게 만들고 설계한 AI Agent에는 아래와 같은 문제가 있다고 생각한다 
- *지나친 의존성*
	-  **순수 함수(Pure Function)**로서의 역할을 상실** : ToolResult 반환 형식에 지나치게 의존적인 비즈니스 로직 
	- 순수 함수 형태를 유지 못하고 곳곳의 코드에서 LangGraph 종속성 노출 
- *하나의 노드(Update_state_node)에서 모든 툴을 처리* ➡ **if-elif 지옥** 

>[!QUESTION] 들었던 의문 
>“툴로 등록한 함수는 정말 AI 전용이어야 할까?  
>아니면 어디서든 호출 가능한 순수 비즈니스 로직이어야 하지 않을까?”

이런 문제 인식과 의문이 프로젝트가 다 끝난 마다에도 리팩토링을 하고 싶은 마음을 불태웠다.

일단 내가 가장 이상적이라 생각하는 그림은 
1. Service는 순수한 도메인 로직만 담당하게 하고 
2. Tool은 Service를 호출하고 일관된 반환타입으로 감싸주는 역할 
3. Node는 Tool실행 결과를 받아 상태를 업데이트하는 단위로 생각했다.

이렇게 한다면 각 기능이 독립적이면서 확장이 가능한 구조라고 생각했다.

이를 달성하기 위해 스프링에서 AOP나 어댑터 패턴을 적용하던 경험이 떠올랐다.  
LangGraph에서도 동일한 방식의 계층 분리와 후처리 라우팅 구조를 만들 수 있을 것 같았다.

따라서, 기존의 무작위적인 툴, graph, 함수 구조를 걷어내고 새로운 패턴 구조를 도입하게 되었다. 

---
## 3.  최종 아키텍쳐 그림 및 핵심 원칙 (Remind)
<div id="Drawing_2025-12-05_0015.57.excalidraw.md2"></div><script>(function(){const InitialData={"type":"excalidraw","version":2,"source":"https://github.com/zsviczian/obsidian-excalidraw-plugin/releases/tag/2.18.0","elements":[{"id":"X3z3-gUWVyLHWUOT5uDmH","type":"rectangle","x":-152.0666097005207,"y":-56.91040039062504,"width":212,"height":125.60000610351562,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a0","roundness":{"type":3},"seed":682187386,"version":115,"versionNonce":2094132685,"isDeleted":false,"boundElements":[{"type":"text","id":"phanobym"},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow"},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow"},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow"},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow"},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow"},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"phanobym","type":"text","x":-120.64457448323554,"y":-26.310397338867226,"width":149.1559295654297,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a1","roundness":null,"seed":1012047078,"version":215,"versionNonce":109998159,"isDeleted":false,"boundElements":[],"updated":1765293195538,"link":null,"locked":false,"text":"Agent_node\n(두뇌)","rawText":"Agent_node\n(두뇌)","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"X3z3-gUWVyLHWUOT5uDmH","originalText":"Agent_node\n(두뇌)","autoResize":true,"lineHeight":1.15},{"id":"OH2497AIboS6cZZRICftD","type":"rectangle","x":-137.66642252604169,"y":138.55628458658856,"width":191.99995930989584,"height":93.066660563151,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a2","roundness":{"type":3},"seed":1759268454,"version":138,"versionNonce":1045974157,"isDeleted":false,"boundElements":[{"type":"text","id":"p5d56VBH"},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow"},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow"},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow"},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow"}],"updated":1765423715410,"link":null,"locked":false},{"id":"p5d56VBH","type":"text","x":-102.20242309570314,"y":168.98961486816407,"width":121.07196044921875,"height":32.199999999999996,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a3","roundness":null,"seed":1077162490,"version":174,"versionNonce":1729728111,"isDeleted":false,"boundElements":[],"updated":1765293195538,"link":null,"locked":false,"text":"Tool Layer","rawText":"Tool Layer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"OH2497AIboS6cZZRICftD","originalText":"Tool Layer","autoResize":true,"lineHeight":1.15},{"id":"YmNG4ENWSaXq9ugMgHRXY","type":"ellipse","x":-134.46634928385419,"y":292.9561564127607,"width":249.86661783854169,"height":163,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a6","roundness":{"type":2},"seed":1924747302,"version":281,"versionNonce":1800229059,"isDeleted":false,"boundElements":[{"type":"text","id":"FFfqsG6c"},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow"},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow"},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow"},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow"},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow"}],"updated":1765423733134,"link":null,"locked":false},{"id":"FFfqsG6c","type":"text","x":-89.04819575157511,"y":342.1269537460571,"width":159.34793090820312,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a7","roundness":null,"seed":1037160294,"version":324,"versionNonce":353514189,"isDeleted":false,"boundElements":[],"updated":1765423715409,"link":null,"locked":false,"text":"route_after_\ntools","rawText":"route_after_tools","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"YmNG4ENWSaXq9ugMgHRXY","originalText":"route_after_tools","autoResize":true,"lineHeight":1.15},{"id":"JtVq8TF3VQFwMa_Am6Wgi","type":"rectangle","x":-296.599873860677,"y":596.4228719075521,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a8","roundness":{"type":3},"seed":1361230246,"version":135,"versionNonce":931090669,"isDeleted":false,"boundElements":[{"type":"text","id":"6PxyvEs3"},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow"},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow"}],"updated":1765423733134,"link":null,"locked":false},{"id":"6PxyvEs3","type":"text","x":-288.6878102620442,"y":633.4228841145833,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"a9","roundness":null,"seed":849849766,"version":197,"versionNonce":174546701,"isDeleted":false,"boundElements":[],"updated":1765423733574,"link":null,"locked":false,"text":"handle Node\n1","rawText":"handle Node 1","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"JtVq8TF3VQFwMa_Am6Wgi","originalText":"handle Node 1","autoResize":true,"lineHeight":1.15},{"id":"ooSxET1T42g_5rrv2Buu6","type":"rectangle","x":-101.71778517502997,"y":601.7255687224559,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aA","roundness":{"type":3},"seed":536348410,"version":165,"versionNonce":679551651,"isDeleted":false,"boundElements":[{"type":"text","id":"USqQBn57"},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow"},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"USqQBn57","type":"text","x":-93.80572157639716,"y":638.7255809294871,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aB","roundness":null,"seed":1343297466,"version":225,"versionNonce":755153891,"isDeleted":false,"boundElements":[],"updated":1765423716359,"link":null,"locked":false,"text":"handle Node\n2","rawText":"handle Node 2","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"ooSxET1T42g_5rrv2Buu6","originalText":"handle Node 2","autoResize":true,"lineHeight":1.15},{"id":"6IA8bLzWF3smACML4jSQf","type":"rectangle","x":113.19502532176489,"y":604.2690867888622,"width":164.00006103515625,"height":138.4000244140625,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aC","roundness":{"type":3},"seed":1110571750,"version":165,"versionNonce":2102176813,"isDeleted":false,"boundElements":[{"type":"text","id":"ctSJUZff"},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow"},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow"},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow"}],"updated":1765423716360,"link":null,"locked":false},{"id":"ctSJUZff","type":"text","x":121.1070889203977,"y":641.2690989958934,"width":148.17593383789062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aD","roundness":null,"seed":624799270,"version":224,"versionNonce":967264131,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"text":"handle Node\n3","rawText":"handle Node 3","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"6IA8bLzWF3smACML4jSQf","originalText":"handle Node 3","autoResize":true,"lineHeight":1.15},{"id":"5Xakb3zyGnOUW3R44YOF0","type":"arrow","x":-44.96001833907274,"y":79.6896057128906,"width":0.3780108839171419,"height":47.866678873697964,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aE","roundness":{"type":2},"seed":1592951034,"version":187,"versionNonce":1282525165,"isDeleted":false,"boundElements":[],"updated":1765423716367,"link":null,"locked":false,"points":[[0,0],[-0.3780108839171419,47.866678873697964]],"lastCommittedPoint":null,"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.5079317665142968,0.5079317665142967]},"endBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.4785926893582332,0.47859268935823324]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"bz9WL9UKeCvZuHt0L6BhE","type":"arrow","x":-35.97674949446286,"y":242.6229451497396,"width":7.71713564728401,"height":40.21517757733281,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aF","roundness":{"type":2},"seed":2066691110,"version":180,"versionNonce":1050717005,"isDeleted":false,"boundElements":[],"updated":1765423733134,"link":null,"locked":false,"points":[[0,0],[7.71713564728401,40.21517757733281]],"lastCommittedPoint":[5.333333333333314,133.3333333333333],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.47450320286991415,0.5254967971300856]},"endBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.49542835966062526,0.5001]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"Vs6QdldLERP8ULSo0kpOF","type":"arrow","x":-93.81644360152464,"y":447.02926860582744,"width":110.27659416758468,"height":138.39360330172468,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aG","roundness":{"type":2},"seed":1846062010,"version":238,"versionNonce":711654189,"isDeleted":false,"boundElements":[],"updated":1765423733137,"link":null,"locked":false,"points":[[0,0],[-110.27659416758468,138.39360330172468]],"lastCommittedPoint":[-238.93326822916663,164.26664225260424],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.39407252956333966,0.5001]},"endBinding":{"elementId":"JtVq8TF3VQFwMa_Am6Wgi","mode":"orbit","fixedPoint":[0.30531257047929755,0.3053125704792968]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"d4tvDhzNOoG21GI3kfStD","type":"arrow","x":-12.347064349429665,"y":466.936333745727,"width":2.236567317393508,"height":123.78923497672895,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aH","roundness":{"type":2},"seed":1943452858,"version":165,"versionNonce":150862861,"isDeleted":false,"boundElements":[],"updated":1765423733135,"link":null,"locked":false,"points":[[0,0],[2.236567317393508,123.78923497672895]],"lastCommittedPoint":[28.79996744791663,167.46663411458326],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.482051947950208,0.5001]},"endBinding":{"elementId":"ooSxET1T42g_5rrv2Buu6","mode":"orbit","fixedPoint":[0.5664038097391358,0.43359619026086504]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"10mnFV_Uqn-bW9mP8Efk9","type":"arrow","x":64.38300120304834,"y":452.08551034168437,"width":142.24318647083456,"height":141.18357644717793,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aI","roundness":{"type":2},"seed":1506914790,"version":169,"versionNonce":1029501549,"isDeleted":false,"boundElements":[],"updated":1765423733135,"link":null,"locked":false,"points":[[0,0],[142.24318647083456,141.18357644717793]],"lastCommittedPoint":[313.59993489583337,177.06669108072924],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.5001,0.5263112548480741]},"endBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[0.8039593463345159,0.19604065366548287]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"E_cqBUHtJbuVFQ0oIN4jH","type":"arrow","x":-305.61116683390895,"y":612.810835820468,"width":165.00685102272652,"height":611.9994610187518,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aJ","roundness":{"type":2},"seed":103747962,"version":391,"versionNonce":963084995,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[-22.46229388933824,-540.8442968492684],[142.54455713338828,-611.9994610187518]],"lastCommittedPoint":[221.53846153846155,-733.5384427584133],"startBinding":{"elementId":"JtVq8TF3VQFwMa_Am6Wgi","mode":"orbit","fixedPoint":[-0.05513419038195911,0.11366915387463353]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.24411731487226143,0.24411731487226132]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"7OXDbWoqyTia7CrJOoDpf","type":"arrow","x":69.27028907568963,"y":612.5183261619767,"width":95.37441175955671,"height":599.4476452992812,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aK","roundness":{"type":2},"seed":333631462,"version":387,"versionNonce":349781741,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[95.37441175955671,-426.0902800293187],[1.663101223789667,-599.4476452992812]],"lastCommittedPoint":[19.692195012019283,-695.3846153846155],"startBinding":{"elementId":"ooSxET1T42g_5rrv2Buu6","mode":"orbit","fixedPoint":[0.8893468411323211,0.889346841132321]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.9041445704193891,0.09585542958061087]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"3JFuNoj451ixdidagOnMV","type":"arrow","x":272.4719744028724,"y":600.712209764589,"width":201.53858410339308,"height":593.5726680637304,"angle":0,"strokeColor":"#1971c2","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aM","roundness":{"type":2},"seed":1858775674,"version":351,"versionNonce":797979235,"isDeleted":false,"boundElements":[],"updated":1765423716360,"link":null,"locked":false,"points":[[0,0],[-76.64826214935698,-493.13544756121803],[-201.53858410339308,-593.5726680637304]],"lastCommittedPoint":[-253.5385366586538,-735.999990609976],"startBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[1.0401707832974996,0.5001]},"endBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8135583943715337,0.18644160562846634]},"startArrowhead":null,"endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"1sQyfb53KVknwBxb3567b","type":"rectangle","x":290.92003498818116,"y":181.50138495464492,"width":199.46663411458326,"height":121.60001627604163,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aN","roundness":{"type":3},"seed":1164082598,"version":198,"versionNonce":730484419,"isDeleted":false,"boundElements":[{"type":"text","id":"SN7dtwUA"},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow"}],"updated":1765423715410,"link":null,"locked":false},{"id":"SN7dtwUA","type":"text","x":347.01537444537513,"y":210.10139309266575,"width":87.27595520019531,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aO","roundness":null,"seed":1106535846,"version":257,"versionNonce":528199139,"isDeleted":false,"boundElements":[],"updated":1765423715409,"link":null,"locked":false,"text":"Service\nLayer","rawText":"Service\nLayer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"1sQyfb53KVknwBxb3567b","originalText":"Service\nLayer","autoResize":true,"lineHeight":1.15},{"id":"tF0uax3CAq_Znji7ZpCY4","type":"arrow","x":65.33353678385416,"y":199.03019314191843,"width":214.586498204327,"height":25.32408157876418,"angle":0,"strokeColor":"#e03131","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aP","roundness":{"type":2},"seed":1239326758,"version":101,"versionNonce":1314524899,"isDeleted":false,"boundElements":[],"updated":1765423715420,"link":null,"locked":false,"points":[[0,0],[214.586498204327,25.32408157876418]],"lastCommittedPoint":[197.33333333333331,2.1333414713541288],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.5186504115874699,0.51865041158747]},"endBinding":{"elementId":"1sQyfb53KVknwBxb3567b","mode":"orbit","fixedPoint":[0.4502438138781704,0.45024381387817053]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"5QKjK6LP1M3apzPJ-pNek","type":"diamond","x":316.6641156841124,"y":349.9826977085189,"width":191.68035831974692,"height":188.53801963434958,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aQ","roundness":{"type":2},"seed":1390001359,"version":132,"versionNonce":389603117,"isDeleted":false,"boundElements":[{"type":"text","id":"Nq2NIrP5"},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow"},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow"},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow"},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow"}],"updated":1765423761103,"link":null,"locked":false},{"id":"Nq2NIrP5","type":"text","x":380.8042293729359,"y":428.0172026171063,"width":63.55995178222656,"height":32.199999999999996,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aR","roundness":null,"seed":506086913,"version":173,"versionNonce":1316561795,"isDeleted":false,"boundElements":[],"updated":1765423755229,"link":null,"locked":false,"text":"State","rawText":"State","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"5QKjK6LP1M3apzPJ-pNek","originalText":"State","autoResize":true,"lineHeight":1.15},{"id":"YQ5rNIULGbw2vQfz3Du0P","type":"diamond","x":285.0336028137207,"y":-29.54062398754101,"width":209.3141097151048,"height":160.2693317148178,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aS","roundness":{"type":2},"seed":1182592833,"version":328,"versionNonce":1161884867,"isDeleted":false,"boundElements":[{"type":"text","id":"SSHfSr2H"},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow"}],"updated":1765423751442,"link":null,"locked":false},{"id":"SSHfSr2H","type":"text","x":346.01415355792653,"y":18.326708941163446,"width":87.69595336914062,"height":64.39999999999999,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#b2f2bb","fillStyle":"solid","strokeWidth":2,"strokeStyle":"solid","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aT","roundness":null,"seed":712990497,"version":384,"versionNonce":594240611,"isDeleted":false,"boundElements":[],"updated":1765423751442,"link":null,"locked":false,"text":"Check\nPointer","rawText":"Check\nPointer","fontSize":28,"fontFamily":7,"textAlign":"center","verticalAlign":"middle","containerId":"YQ5rNIULGbw2vQfz3Du0P","originalText":"Check\nPointer","autoResize":true,"lineHeight":1.15},{"id":"s0dfDhxtFz2bhkL1Bbz2s","type":"arrow","x":282.3718527171511,"y":611.5231455969616,"width":83.87264672259482,"height":103.0157766932067,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aW","roundness":{"type":2},"seed":2075619841,"version":45,"versionNonce":76444045,"isDeleted":false,"boundElements":[],"updated":1765423761103,"link":null,"locked":false,"points":[[0,0],[83.87264672259482,-103.0157766932067]],"startBinding":{"elementId":"6IA8bLzWF3smACML4jSQf","mode":"orbit","fixedPoint":[0.6274054152117697,0.6274054152117706]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.5001,0.539323457812219]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"UYZG6RZ38cPO9fjBvABCY","type":"arrow","x":125.94297844853571,"y":382.0371151229574,"width":188.18520072945142,"height":53.118621621951036,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aX","roundness":{"type":2},"seed":186865359,"version":141,"versionNonce":456112931,"isDeleted":false,"boundElements":[],"updated":1765423755229,"link":null,"locked":false,"points":[[0,0],[188.18520072945142,53.118621621951036]],"startBinding":{"elementId":"YmNG4ENWSaXq9ugMgHRXY","mode":"orbit","fixedPoint":[0.9294429416153576,0.5001]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.15346942903874886,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"iZq-YCBwmN89ucnMZg8hs","type":"arrow","x":65.20922904997465,"y":212.03847310280614,"width":249.56855020162558,"height":221.90322301622874,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aY","roundness":{"type":2},"seed":1049835279,"version":322,"versionNonce":866712845,"isDeleted":false,"boundElements":[],"updated":1765423758961,"link":null,"locked":false,"points":[[0,0],[249.56855020162558,221.90322301622874]],"startBinding":{"elementId":"OH2497AIboS6cZZRICftD","mode":"orbit","fixedPoint":[0.7551440573671417,0.24485594263285798]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.05076345491501994,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"LYalxgUXtK-2IFIBPxBeD","type":"arrow","x":70.84447803383041,"y":40.83692118305623,"width":245.86966452222515,"height":390.0743360198682,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aZ","roundness":{"type":2},"seed":1568001935,"version":368,"versionNonce":812801731,"isDeleted":false,"boundElements":[],"updated":1765423755230,"link":null,"locked":false,"points":[[0,0],[245.86966452222515,390.0743360198682]],"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8246251504386163,0.1753748495613837]},"endBinding":{"elementId":"5QKjK6LP1M3apzPJ-pNek","mode":"orbit","fixedPoint":[0.044097163342550524,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false},{"id":"jHuKO11SEqqHq-W3o9NiF","type":"arrow","x":70.93339029947929,"y":-18.43852436629765,"width":211.5780104038157,"height":61.96825371528639,"angle":0,"strokeColor":"#1e1e1e","backgroundColor":"#eebefa","fillStyle":"solid","strokeWidth":2,"strokeStyle":"dotted","roughness":1,"opacity":100,"groupIds":[],"frameId":null,"index":"aa","roundness":{"type":2},"seed":1880908577,"version":210,"versionNonce":501669891,"isDeleted":false,"boundElements":[],"updated":1765423751443,"link":null,"locked":false,"points":[[0,0],[211.5780104038157,61.96825371528639]],"startBinding":{"elementId":"X3z3-gUWVyLHWUOT5uDmH","mode":"orbit","fixedPoint":[0.8109894317567161,0.18901056824328386]},"endBinding":{"elementId":"YQ5rNIULGbw2vQfz3Du0P","mode":"orbit","fixedPoint":[0.10278159774802884,0.5001]},"startArrowhead":"arrow","endArrowhead":"arrow","elbowed":false,"moveMidPointsWithElement":false}],"appState":{"theme":"light","viewBackgroundColor":"#ffffff","currentItemStrokeColor":"#1e1e1e","currentItemBackgroundColor":"#eebefa","currentItemFillStyle":"solid","currentItemStrokeWidth":2,"currentItemStrokeStyle":"dotted","currentItemRoughness":1,"currentItemOpacity":100,"currentItemFontFamily":6,"currentItemFontSize":28,"currentItemTextAlign":"left","currentItemStartArrowhead":"arrow","currentItemEndArrowhead":"arrow","currentItemArrowType":"round","currentItemFrameRole":null,"scrollX":971.788479447701,"scrollY":224.94900663804094,"zoom":{"value":1},"currentItemRoundness":"round","gridSize":20,"gridStep":5,"gridModeEnabled":false,"gridColor":{"Bold":"rgba(217, 217, 217, 0.5)","Regular":"rgba(230, 230, 230, 0.5)"},"currentStrokeOptions":null,"frameRendering":{"enabled":true,"clip":true,"name":true,"outline":true,"markerName":true,"markerEnabled":true},"objectsSnapModeEnabled":false,"activeTool":{"type":"selection","customType":null,"locked":false,"fromSelection":false,"lastActiveTool":null},"disableContextMenu":false},"files":{}};InitialData.scrollToContent=true;App=()=>{const e=React.useRef(null),t=React.useRef(null),[n,i]=React.useState({width:void 0,height:void 0});return React.useEffect(()=>{i({width:t.current.getBoundingClientRect().width,height:t.current.getBoundingClientRect().height});const e=()=>{i({width:t.current.getBoundingClientRect().width,height:t.current.getBoundingClientRect().height})};return window.addEventListener("resize",e),()=>window.removeEventListener("resize",e)},[t]),React.createElement(React.Fragment,null,React.createElement("div",{className:"excalidraw-wrapper",ref:t},React.createElement(ExcalidrawLib.Excalidraw,{ref:e,width:n.width,height:n.height,initialData:InitialData,viewModeEnabled:!0,zenModeEnabled:!0,gridModeEnabled:!1})))},excalidrawWrapper=document.getElementById("Drawing_2025-12-05_0015.57.excalidraw.md2");ReactDOM.render(React.createElement(App),excalidrawWrapper);})();</script>

1. **Service는 순수 비즈니스 로직** - LangChain/LangGraph에 대해 전혀 모름
2. **Tool은 얇은 어댑터** - Service 호출 + ToolResult 포장만 담당
3. **Node는 상태 관리** - Tool 결과를 받아서 State 업데이트

---
## 4.  계층별 책임

MateTrip AI Agent를 이루는 3계층 구조를 어떻게 설계했고 분리했는지에 대한 설명
**목적 : 도구-비즈니스로직-상태관리분리 등을 위해 설계**되었다

---
### 4.1.  Layer 1: Service (순수 비즈니스 로직)
> Service 계층은 프로젝트 전체에서 가장 중요한 원칙인 **순수성** 을 기반으로 설계

- LangChain, LangGraph와 완전히 독립적이다.⭐⭐
- ToolResult같은 반환형식이나 LangChain Message를 다루지 않는다.
- **DB 조회 → 도메인 로직 처리 → DTO 반환**이라는 가장 기본적인 책임만 가진다
- **재사용성**: REST API, CLI, 테스트 코드 등 어디서든 사용 가능

즉, Service 레이어는 "AI 기능을 위한 코드"가 아니라 프로젝트 전체의 비즈니스 로직을 담당하는 레이어로 설계해야 한다.

아래 예시는 DTO기반의 요청-응답 구조 
```python
class PlaceService:
    def find_replacement_places(
        self, request: ReplacePlaceRequest
    ) -> List[NearbyPlaceResponse]:
        """
        순수 비즈니스 로직 - ToolResult 모름

        ReplacePlaceRequest(6개 필드)
        - latitude, longitude, replace_count, excluded_place_ids,
          category, radius_km
        -> ReplacePlaceRequest DTO
        """
        places = self.repository.find_places_within_radius(
            latitude=request.latitude,
            longitude=request.longitude,
            radius_km=request.radius_km,
            category=request.category,
            limit=request.replace_count,
            excluded_place_ids=request.excluded_place_ids,
        )
        return [NearbyPlaceResponse.from_entity(p) for p in places]
```

핵심은 Service는 LangGraph의 존재를 모른다는점 
이 덕분에, Service는 유지보수성, 가독성, 테스트가능성, 재사용성이 모두 확보된다.

---
### 4.2.  Layer 2: Tool (LLM 어댑터)

> **얇은 래퍼 클래스**: 비즈니스 로직 없음, 변환만 수행

Tool은 LangGraph가 호출하는 함수이다. 따라서 LangGraph와 내부 서비스를 연계하는 중간 다리 역할을 담당한다.

여기에는 비즈니스 로직이 들어가면 안되고 아래의 3가지 역할을 위주로 하도록 설계했다.
1. LLM으로부터 전달받은 파라미터 검증
2. Service 호출을 위한 DTO 정규화
3. Service 결과 정규화 및 메타데이터 추가 

예시 
```python
@tool
def replace_places(replace_target_ids, latitude, longitude, ...):
    """LLM 어댑터 - Service 호출 + ToolResult 포장"""

    # 1) LLM에서 들어온 파라미터를 DTO로 캡슐화 
    request = ReplacePlaceRequest.create(
        latitude=latitude,
        longitude=longitude,
        replace_count=len(replace_target_ids),
        excluded_place_ids=excluded_place_ids,
        category=mapped_category,
        radius_km=radius_km,
    )

    # Service Layer 호출 (순수 비즈니스 로직)
    place_responses = PlaceService(db).find_replacement_places(request)

    # ToolResult로 감싸기 (일관된 JSON 스키마)
    return ToolResult(
        success=True,
        data=PlaceRecommendationData(
            places=[p.model_dump() for p in place_responses],
            replaced_place_ids=replace_target_ids,  # 메타정보
        ),
        # 메타 정보들 SUCCESS, replace_id, error_code 등 공통 필드 확장 
    ).model_dump()
```
Tool은 진짜 로직은 없고 단순 전달 및 결과를 변환해주는 어뎁터 역할

*❤️장점*
- Service 로직 오염 안시킬 수 있다.
- Service 결과 표준화 가능 ➡ LLM이 이해하기 좋은 일관된 형태의 반환값 유지 가능
- Tool은 매우 얇아서 확장/변경이 쉬움

---
### 4.3.  Layer 3: Node (상태 관리)
> Tool 실행 결과를 기반으로 상태를 업데이트하는 계층 

Tool이 “데이터를 가져오는 역할”이라면 Node는 “그 데이터를 AgentState에 반영하는 역할”을 한다.

LangGraph에서는 ~~ 하면서 상태를 변경한다. AgentState변경하면서 

Node Layer의 역할
- `ToolResult`를 파싱하고
- `AgentState`에 반영하고
- 다시 Agent 노드로 돌아가는 흐름을 만든다

**파일 구조**:
```
app/agent/nodes/
├── handle_replace_places_node.py       # replace_places 전용
├── handle_place_recommendation_node.py # 일반 장소 추천 전용(여럿)
├── handle_travel_route_node.py         # create_travel_route 전용
└── ...
```

**예시**:
```python
def handle_replace_places_node(state: AgentState) -> AgentState:
    """replace_places Tool 결과를 받아서 상태 업데이트"""

    # 1. ToolResult 파싱
    data = get_last_tool_message(state["messages"]).content["data"]
    replaced_ids = data["replaced_place_ids"]
    new_places = data["places"]

    # 2. 기존 장소에서 제거
    current = state["last_recommended_places"]
    updated = _drop_places_by_ids(current, replaced_ids)

    # 3. 새 장소 추가
    updated.extend(to_simple_places(new_places))

    return {"last_recommended_places": updated}
```


>[!tip] 
이 계층이 등장하면서 기존의 `update_state_node` 하나에 모든 Tool 분기 로직이 모이는” 구조를 완전히 제거할 수 있었다.

---
### 4.4.  Router - 후처리 노드를 결정하는 분기 
> Tool 실행이 끝난 후 어떤 Node를 호출할지 결정하는 함수를 설정하는 것 <br>
> `route_after_tools()`가 Tool별로 적절한 후처리 노드로 라우팅

#### 4.4.1.  기존 문제점 : Before 💢
```python
# update_state_node.py - 모든 Tool 처리
def update_state_node(state):
    if tool_name == "replace_places":
        # 특수 로직 1
    elif tool_name == "create_travel_route":
        # 특수 로직 2
    else:
        # 일반 처리
```

- Tool이 늘어날 때마다 `if`문 추가
- 단일 파일에 모든 로직 집중
- Tool이 ToolResult 반환 → 순수하지 않음


#### 4.4.2.  개선 : After ✅
```python
# Tool -> 후처리 노드 매핑 (중앙 집중 관리)
TOOL_POSTPROCESSING_ROUTES: dict[Hashable, str] = {
    "replace_places": "handle_replace_places",
    "recommend_nearby_places": "handle_place_recommendation",
    "recommend_popular_places_in_region": "handle_place_recommendation",
    "recommend_places_by_all_users": "handle_workspace_recommendation",
    "create_travel_route": "handle_travel_route",
}
```
- Tool 이름 ➡ Node 이름으로 매핑 
- 기존에 상태 업데이트를 처리하던 node인 `Update_state_node`에 분기 되어있던 로직들이 분리되어 유지보수 용이

Tool 실행 후 `route_after_tools()`
```python
# Graph만들 때, workflow.add_conditional_edges("tools", route_after_tools, postprocess_edges)에서 사용한다 
def route_after_tools(state: AgentState) -> str:
    """
    Tool 실행 후 어떤 후처리 노드로 보낼지 결정
    각 Tool은 전용 후처리 노드가 상태 변경을 담당
    예시
    - replace_places → handle_replace_places
    - recommend_* → handle_place_recommendation
    - create_travel_route → handle_travel_route
    """
    last_tool_message = get_last_tool_message(state.get("messages", []))
    if not last_tool_message:
        return "agent"

    tool_name = getattr(last_tool_message, "name", "")
    target_node = TOOL_POSTPROCESSING_ROUTES.get(tool_name)
    if not target_node:
        return "agent"

    return target_node
```

---
### 4.5.  정리 

1. Service는 순수 비즈니스 로직 
2. Tool은 LLM 어댑터 
3. Node는 상태 관리 (Tool별 분리)
4. 명시적 라우팅 (route_after_tools)

---
## 5.  LangGraph 에이전트 만드는 예시 

`create_agent_graph` - LangGraph 에이전트 만드는 함수 
```PYTHON
# =========================
# 그래프 구성 (후처리 노드 분리 패턴)
# =========================
def create_agent_graph():
    """
    LangGraph 생성 - 후처리 노드 분리 패턴
    구조:
    1. Router → Agent → Tools (도구 실행)
    2. Tools → route_after_tools (라우터)
    3. route_after_tools → 각 Tool별 전용 후처리 노드
       - replace_places → handle_replace_places
       - recommend_* → handle_place_recommendation
       - create_travel_route → handle_travel_route
       - 기타 → agent (바로 복귀)
    4. 후처리 노드 → agent (상태 업데이트 후 복귀)
    """
		workflow = StateGraph(AgentState)
		
		# 노드 추가 
		# 엣지 추가 
    
    # tools -> route_after_tools (Tool별 후처리 노드로 분기)
    postprocess_edges: dict[Hashable, str] = {
        node: node for node in TOOL_POSTPROCESSING_ROUTES.values()
    }
    postprocess_edges["agent"] = "agent"  # 기본 경로
    workflow.add_conditional_edges("tools", route_after_tools, postprocess_edges)
    
    # todo : 하드코딩 부분 수정 
		# 각 후처리 노드 -> agent (상태 업데이트 후 에이전트로 복귀)
    workflow.add_edge("handle_replace_places", "agent")
    workflow.add_edge("handle_place_recommendation", "agent")
    workflow.add_edge("handle_workspace_recommendation", "agent")
    workflow.add_edge("handle_travel_route", "agent")
    
    memory = MemorySaver()
    return workflow.compile(checkpointer=memory)    
```

---
## 6.  이 패턴의 장점 정리 

### 6.1.  책임 분리
- **Service**: "데이터를 어떻게 가져올까?"
- **Tool**: "LLM에게 데이터를 어떻게 전달할까?"
- **Node**: "가져온 데이터로 상태를 어떻게 바꿀까?"

---
### 6.2.  명시성 
- Tool별 분기가 `route_after_tools()`에 명시적으로 표현
- `if tool_name == ...` 분기문이 여러 곳에 흩어지지 않음

---
## 7.  예시: replace_places 플로우

```
1. LLM이 replace_places Tool 호출 결정
   ↓
2. Tool Layer (@tool replace_places)
   - ReplacePlaceRequest DTO 생성 (6개 파라미터 캡슐화)
   - Service.find_replacement_places(request) 호출
   - ToolResult로 포장 (replaced_place_ids 메타정보 추가)
   ↓
3. route_after_tools() 라우터
   - tool_name == "replace_places" 확인
   - "handle_replace_places" 노드로 라우팅
   ↓
4. Node Layer (handle_replace_places_node)
   - ToolResult 파싱
   - 기존 장소에서 교체 대상 제거
   - 새 장소 추가
   - AgentState 업데이트
   ↓
5. Agent로 복귀
```

---
## 8.  번외 : 아직 남아있는 리팩토링 대상들 

아직도 코드가 몇개 맘에 안든다. 
특히 graph생성에서 노드명, 엣지명 등을 입력하는 과정에서 나는 여전히 하드코딩되어 있다. 고치고 싶은데.. 이제 더 이상 코드를 고치고 PR올리면 팀원들도 불안해 할 것 같기도 하고(다 끝난 마당에 왜 코드를 또 고치냐...이런...) 나도 지금 이력서 쓰고, 이전에 했던거 정리하고 면접, 코테 준비 등으로 바쁘다보니 이걸로 만족해야 할 것 같다.
![Pasted image 20251204153622.png](/img/user/supporter/image/Pasted%20image%2020251204153622.png)
AI Agent 아키텍처 이게 맞는지 모르겠다.
근데 이전에 주먹구구식으로 LangGraph관련 코드들을 작성하기 시작하면서 점점 코드를 개선해 나가는 과정이 좋은 경험이였던 것 같다. 

---
## 9.  번외2 - 어뎁터 패턴 쓰면서 느낀 한계와 부작용


- 이번 아키텍처에서 Tool Layer를 **“LLM 어댑터”** 로 두고 3계층을 강하게 분리한 건 분명 장점이 있었다.
- 하지만 막상 프로젝트 전체에 적용해보니, **“무조건 좋은 패턴”이라고 말하기 어렵다**는 생각이 들었다.

### 9.1.  복잡하고 번거로움 
작은 기능하나를 추가해도 Service - Tool - Node -Router 까지 다 건드려야 했다.
레이어가 너무 많다. 이러한 복잡성은 Agent Graph를 그리는 것에도 영향을 준다 
특히나 이번 프로젝트처럼 긴박한 조건이라면 더욱 더 불필요한 패턴이었던 것 같다.
동선이 너무 길어서 디버깅도 귀찮고....

---
### 9.2.  ToolResult 스키마로 인한 강한 결합

한 번 `ToolResult` 스키마를 정의해두니까 중간에 필드를 바꾸거나 구조를 바꾸기 어려워졌다. 중간에 ToolResult스키마 변경할만한게 있나? 생각할 때 변경을 생각했는데 Node에서 `data["..."]`를 파싱하는 코드가 여러 군데 존재하여 변경도 어렵게 느꼈다. <br>
결국, 표준화를 하다보니 동시에 “강한 결합”이 동반하게 되어 썩... 맘에 드는 코드가 된 것 같지는 않다.



