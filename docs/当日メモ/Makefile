# 変数定義(競技開始時に変更)----------------------
# 例として、ISUCON12のときの設定にしてある
SERVICE_NAME=isuports.service

DB_NAME=isuports
DB_ROOT_USER=root
DB_ROOT_PASSWORD=root

RUBY_APP_FILE_NAME=app.rb
RUBY_WORKING_PATH=/home/isucon/webapp/ruby
ESTACKPROF_INPUT_PATH=/home/isucon/webapp/ruby/tmp/
ESTACKPROF_OUTPUT_PATH:=/home/isucon/webapp/ruby/tmp/log_$(shell date +'%Y%m%d%H%M').txt

ACCESS_LOG=/var/log/nginx/access.log

MYSQL_SLOW_QUERY_LOG=/var/log/mysql/mysql-slow.log
MYSQL_ERROR_LOG=/var/log/mysql/error.log
SLOW_QUERY_DUMP_OUTPUT_PATH:=/home/isucon/log/slow_query/slow_query_log_$(shell date +'%Y%m%d%H%M').txt

ALP_CONFIG_PATH=/home/isucon/webapp/alp_config.yml
ALP_OUTPUT_PATH:=/home/isucon/log/alp/alp_log_$(shell date +'%Y%m%d%H%M').txt

ALL_SAVE_LOGS_PATH=/home/isucon/webapp/all_logs
# インストールコマンド----------------------
.PHONY: install-tools
install-tools:
	sudo apt update
	sudo apt install -y dstat unzip git wget tree percona-toolkit fish htop

.PHONY: install-alp
install-alp: # 実行前にALP_CONFIG_PATHを確認し、ログ保存先のディレクトリが存在するか確認すること
	mkdir -p ~/log/alp && \
	cd ~/log/alp && \
	wget https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.zip && \
	unzip alp_linux_amd64.zip && \
	sudo install ./alp /usr/local/bin && \
	touch ${ALP_CONFIG_PATH} && \
	echo "alpのバージョン確認"
	alp --version

.PHONY: install-query-digest
install-query-digest:
	mkdir -p ~/tmp && \
	cd ~/tmp/ && \
	git clone git@github.com:kazeburo/query-digester.git && \
	cd query-digester && \
	sudo install query-digester /usr/local/bin
	echo "TODO: query-digesterのパスチェック"
	which query-digester

# 汎用コマンド----------------------
.PHONY: login-mysql
login-mysql:
	mysql -u${DB_ROOT_USER} -p${DB_ROOT_PASSWORD} ${DB_NAME}

.PHONY: restart
restart:
	sudo systemctl daemon-reload
	sudo systemctl restart $(SERVICE_NAME)
	sudo systemctl restart mysql.service
	sudo systemctl restart nginx.service
	# sudo systemctl restart mariadb.service # mariadbの場合

.PHONY: check-service
check-service:
	sudo journalctl -fu $(SERVICE_NAME)

.PHONY: deploy-service
deploy-service: delete-input-log restart
	@echo "古い入力データを削除し、サービスを再起動しました。"

.PHONY: all-save-logs-and-delete-old-data
all-save-logs-and-delete-old-data: all-save-logs delete-input-log
	@echo "ログをまとめて保存し、古い入力データを削除しました。"

.PHONY: delete-input-log
delete-input-log:
	@echo nginxのログを削除
	@sudo rm /var/log/nginx/access.log
	@sudo systemctl restart nginx
	@find ${ESTACKPROF_INPUT_PATH} -name 'stackprof-*' -exec rm -rf {} \;
	@echo mysqlのログを削除
	@sudo rm ${MYSQL_SLOW_QUERY_LOG}
	@sudo rm ${MYSQL_ERROR_LOG}
	@sudo systemctl restart mysql
	# @sudo systemctl restart mariadb.service # mariadbの場合

.PHONY: all-save-logs
all-save-logs: check-access-log check-estackprof-list check-slow-query-log
	@cd ${RUBY_WORKING_PATH} && \
	LOG_DIR=${ALL_SAVE_LOGS_PATH}/log_$(shell date +'%Y%m%d%H%M')_$$(git rev-parse HEAD) && \
	mkdir -p $${LOG_DIR} && \
	cp ${ALP_OUTPUT_PATH} $${LOG_DIR}/access-log.txt && \
	cp ${ESTACKPROF_OUTPUT_PATH} $${LOG_DIR}/estackprof-list.txt && \
	cp ${SLOW_QUERY_DUMP_OUTPUT_PATH} $${LOG_DIR}/slow-query.txt && \
	echo "---------------------------" && \
	echo "$${LOG_DIR} に保存しました"

# ログ: ベンチマークを動かしながら使うコマンド----------------------
.PHONY: check-cpu
check-cpu:
	dstat -vt

.PHONY: check-memory
check-memory:
	dstat -t -gs --mem --vm --ipc

.PHONY: check-top-cpu
check-top-cpu:
	dstat -ta --top-cpu

.PHONY: check-top-mem
check-top-mem:
	dstat -ta --top-mem

.PHONY: check-access-log
check-access-log: # alpの設定ファイルを事前に$ALP_CONFIG_PATHに用意しておくこと
	sudo alp ltsv --config ${ALP_CONFIG_PATH} > ${ALP_OUTPUT_PATH}
	@echo "必要に応じてアクセスログを削除すること(make delete-access-log)"

.PHONY: check-query-digester
check-query-digester:
	@echo "事前にMySQLにログインして、下記のようにしてパスワードを消す必要あり"
	@echo "mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '';"
	@echo "login-mysqlコマンドを書き換える: mysql -u${DB_ROOT_USER} {DB_NAME}"
	@echo "---------------------------"
	sudo query-digester -duration 10

# ログ: ベンチマークを動かし終わった後に使うコマンド----------------------
.PHONY: check-estackprof-list
check-estackprof-list:
	cd ${RUBY_WORKING_PATH} && \
	bundle exec estackprof list -f ${RUBY_APP_FILE_NAME} > ${ESTACKPROF_OUTPUT_PATH}
	echo "出力先: ${ESTACKPROF_OUTPUT_PATH}"

.PHONY: check-estackprof-top
check-estackprof-top:
	cd ${RUBY_WORKING_PATH} && \
	bundle exec estackprof top -p ${RUBY_APP_FILE_NAME} > ${ESTACKPROF_OUTPUT_PATH}
	echo "出力先: ${ESTACKPROF_OUTPUT_PATH}"

.PHONY: check-estackprof-function
check-estackprof-function:
	@echo "引数として関数名を渡す(例: make check-estackprof-function ARG_FUNCTION=method_name)"
	cd ${RUBY_WORKING_PATH} && \
	bundle exec estackprof top -p ${ARG_FUNCTION} > ${ESTACKPROF_OUTPUT_PATH}
	echo "出力先: ${ESTACKPROF_OUTPUT_PATH}"

.PHONY: check-slow-query-log
check-slow-query-log:
	@echo "MySQLのslow_query_logを有効にしておくこと"
	mkdir -p $(dir ${SLOW_QUERY_DUMP_OUTPUT_PATH})
	sudo mysqldumpslow /var/log/mysql/mysql-slow.log > ${SLOW_QUERY_DUMP_OUTPUT_PATH}

# 当日追加したコマンド一覧----------------------
