1.создал две виртуальные машины и развернул kubernetes на основе containerd + crictl + сеть calico  
ps можно было использовать minicube  
2. создание postgresql  
 -ConfigMap с переменными  
 -PV и PVC указывающие на локальную директорию  
 -deployment postgres  
 -NodePort  с пробросом выделенного порта снаружи.  
 
3.удалив поду, кубернетес создаст новую. данные при этом не теряются.  
 

недостатки.   
-директория локальная. поду надо "прибить" к этой ноде. в продакш варианте использовать схд с линками к каждой ноде.  
