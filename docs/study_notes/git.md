## ȫ������

git config --global user.name xxx������ȫ���û�������Ϣ��¼��~/.gitconfig�ļ���

git config --global user.email xxx@xxx.com������ȫ�������ַ����Ϣ��¼��~/.gitconfig�ļ���

git init������ǰĿ¼���ó�git�ֿ⣬��Ϣ��¼�����ص�.git�ļ�����

## ��������

git add XX ����XX�ļ���ӵ��ݴ���

git commit -m "���Լ����ı�ע��Ϣ"�����ݴ����������ύ����ǰ��֧
git status���鿴�ֿ�״̬

git log���鿴��ǰ��֧�����а汾

git push -u (��һ����Ҫ-u�Ժ���Ҫ) ������ǰ��֧���͵�Զ�ֿ̲�

git clone git@git.xxx.com:xxx/XXX.git����Զ�ֿ̲�XXX���ص���ǰĿ¼��

git branch���鿴���з�֧�͵�ǰ������֧

## �鿴����

git diff XX���鿴XX�ļ�������ݴ����޸�����Щ����

git status���鿴�ֿ�״̬

git log���鿴��ǰ��֧�����а汾

git log --pretty=oneline����һ������ʾ

git reflog���鿴HEADָ����ƶ���ʷ���������ع��İ汾��

git branch���鿴���з�֧�͵�ǰ������֧

git pull ����Զ�ֿ̲�ĵ�ǰ��֧�뱾�زֿ�ĵ�ǰ��֧�ϲ�

## ɾ������

git rm --cached XX�����ļ��Ӳֿ�����Ŀ¼��ɾ������ϣ����������ļ�

git restore --staged xx��==��xx���ݴ������Ƴ�==

git checkout �� XX��git restore XX��==��XX�ļ���δ�����ݴ������޸�ȫ������==

## ����ع�
git reset --hard HEAD^ ��git reset --hard HEAD~ ���������ع�����һ���汾

git reset --hard HEAD^^�����ϻع����Σ��Դ�����

git reset --hard HEAD~100�����ϻع�100���汾

git reset --hard �汾�ţ��ع���ĳһ�ض��汾

## Զ�ֿ̲�
git remote add origin git@git.acwing.com:xxx/XXX.git�������زֿ������Զ�ֿ̲�

git push -u (��һ����Ҫ-u�Ժ���Ҫ) ������ǰ��֧���͵�Զ�ֿ̲�

git push origin
branch_name�������ص�ĳ����֧���͵�Զ�ֿ̲�

git clone git@git.acwing.com:xxx/XXX.git����Զ�ֿ̲�XXX���ص���ǰĿ¼��

git push --set-upstream origin branch_name�����ñ��ص�branch_name��֧��ӦԶ�ֿ̲��branch_name��֧

git push -d origin branch_name��ɾ��Զ�ֿ̲��branch_name��֧

git checkout -t origin/branch_name ��Զ�̵�branch_name��֧��ȡ������

git pull ����Զ�ֿ̲�ĵ�ǰ��֧�뱾�زֿ�ĵ�ǰ��֧�ϲ�

git pull origin branch_name����Զ�ֿ̲��branch_name��֧�뱾�زֿ�ĵ�ǰ��֧�ϲ�

git branch --set-upstream-to=origin/branch_name1 branch_name2����Զ�̵�branch_name1��֧�뱾�ص�branch_name2��֧��Ӧ

## ��֧����

git branch branch_name�������·�֧

git branch���鿴���з�֧�͵�ǰ������֧

git checkout -b branch_name���������л���branch_name�����֧

git checkout branch_name���л���branch_name�����֧

git merge branch_name������֧branch_name�ϲ�����ǰ��֧��

git branch -d branch_name��ɾ�����زֿ��branch_name��֧

git push --set-upstream origin branch_name�����ñ��ص�branch_name��֧��ӦԶ�ֿ̲��branch_name��֧

git push -d origin branch_name��ɾ��Զ�ֿ̲��branch_name��֧

git checkout -t origin/branch_name ��Զ�̵�branch_name��֧��ȡ������

git pull ����Զ�ֿ̲�ĵ�ǰ��֧�뱾�زֿ�ĵ�ǰ��֧�ϲ�

git pull origin branch_name����Զ�ֿ̲��branch_name��֧�뱾�زֿ�ĵ�ǰ��֧�ϲ�

git branch --set-upstream-to=origin/branch_name1 branch_name2����Զ�̵�branch_name1��֧�뱾�ص�branch_name2��֧��Ӧ

## stash�ݴ�
git stash�������������ݴ�������δ�ύ���޸Ĵ���ջ��

git stash apply����ջ���洢���޸Ļָ�����ǰ��֧������ɾ��ջ��Ԫ��

git stash drop��ɾ��ջ���洢���޸�

git stash pop����ջ���洢���޸Ļָ�����ǰ��֧��ͬʱɾ��ջ��Ԫ��

git stash list���鿴ջ������Ԫ��