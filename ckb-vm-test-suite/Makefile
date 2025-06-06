ckb-vm:
	git clone https://github.com/nervosnetwork/ckb-vm ../deps/ckb-vm
	ln -s ../deps/ckb-vm .

report-bench.txt:
	echo -n > $@
	echo -n "os   : " >> $@
	uname >> $@
	echo -n "arch : " >> $@
	uname -m >> $@
	echo -n "cpu  : " >> $@
	cat /proc/cpuinfo | grep name | cut -d: -f2 | cut -c2- | uniq >> $@
	echo "--------------------------------------------------------------------------------" >> $@
	make -C programs report >> $@
	echo "--------------------------------------------------------------------------------" >> $@
	cargo bench >> $@
