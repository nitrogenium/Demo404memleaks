

## RRLaravel memory leak demo

make install
make up


No leaks:

 ab -c 1 -t 120 http://localhost:4001/


Leaks:

 ab -c 1 -t 120 http://localhost:4001/404


