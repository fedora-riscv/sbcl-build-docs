# 序
在README中提到过，sbcl在2.2.2版本能正常编译，2.2.3版本开始出现问题，因而这里对2.2.2到2.2.3的所有patch采取二分法排错

# 脚本
```bash
#!/bin/bash
set -ex

begin_tag="sbcl-2.2.2"
end_tag="sbcl-2.2.3"
ssh_host="root@192.168.122.153"
ssh_sbcl_path="/root/sbcl"
test_filename="sbcl-run-program-test.lisp"
test_file_path="../"
cross_make_patch_path="../sbcl-cross-make.diff"

commit_list=$(git log --pretty=format:"%h" $begin_tag...$end_tag)
commit_length=$(echo "$commit_list" | wc -l)
echo "commit_length=$commit_length"

#begin_index=1
#end_index=$commit_length
begin_index=107
end_index=107
while [[ $begin_index -le $end_index ]]
do
        # Cleaning
        ./clean.sh
        git reset --hard
        git clean -f -x
        ssh $ssh_host cd $ssh_sbcl_path \; ./clean.sh \; git reset --hard \; git clean -f -x

        # Switch commit
        pivot=$(((begin_index+end_index)/2))
        current_hash=$(echo "$commit_list" | head -n $pivot | tail -n 1)
        echo "[$(date)] Current commit hash: $current_hash"
        echo "$(git log -n 1 $current_hash)"
        git checkout $current_hash

        # Logging
        echo "[`date`] [$begin_index - $end_index] pivot: $pivot $current_hash"
        echo "[`date`] [$begin_index - $end_index] pivot: $pivot $current_hash" >> ~/sbcl-half-logs

        # Patching
        patch < $cross_make_patch_path

        # Building
        if ! sh cross-make.sh sync $ssh_host /root/sbcl SBCL_ARCH=riscv64 CFLAGS="-fsigned-char"; then
                echo "[$(date)] compilation failed! Shrinking the range"
                echo "[$(date)] [$pivot] $current_hash FAILED: compilation" >> ~/sbcl-half-logs
                begin_index=$((pivot+1))
                continue
        fi

        # Testing
        scp $test_file_path$test_filename $ssh_host:$ssh_sbcl_path
        ssh $ssh_host cd $ssh_sbcl_path '&&' cat $test_filename \| ./run-sbcl.sh
        scp $ssh_host:$ssh_sbcl_path/output.txt . || echo "[$(date)] Failed to scp output.txt!"
        echo "[$(date)] File content: $(cat output.txt)"
        if [[ $(cat output.txt) == "114514" ]]; then
                echo "[$(date)] $current_hash passed the test! Extending the range"
                echo "[$(date)] [$pivot] $current_hash SUCCEEDED" >> ~/sbcl-half-logs
                end_index=$((pivot-1))
        else
                echo "[$(date)] failed the test! Shrinking the range"
                echo "[$(date)] [$pivot] $current_hash FAILED: run-program" >> ~/sbcl-half-logs
                begin_index=$((pivot+1))
        fi
done
```

# 结论
最后排查出来是[该commit](https://sourceforge.net/p/sbcl/sbcl/ci/e50eda73c)触发的bug，但由于该commit涉及到并发，可能超出本人目前能力范围，目前希望能够得到上游的协助
