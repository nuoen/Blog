网卡设置为监控模式
sudo ip link set wlx3476c53ad55b down
sudo iw wlx3476c53ad55b set type monitor
sudo ip link set wlx3476c53ad55b up