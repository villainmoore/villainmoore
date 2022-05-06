import logging

import conf
from dingtalkchatbot.chatbot import DingtalkChatbot


class DingTalkNotifier(object):

    def __init__(self, payload: dict):
        self.payload = payload
        self.action = self.payload.get('action')
        self.action_prep = 'to' if self.action in ('created', 'opened', 'submitted', None) else 'of'
        self.sender = sender = payload.get('sender') or {}
        self.sender_full_name = sender.get('login')
        self.sender_page = sender.get('html_url')
        self._md_sender = f'[{self.sender_full_name}]({self.sender_page})'
        self.repo = repo = payload.get('repository') or {}
        self.repo_full_name = repo.get('full_name')
        self.repo_page = repo.get('html_url')
        self.repo_language = repo.get('language')
        self.repo_star_count = repo.get('stargazers_count')
        self._md_repo = f'[{self.repo_full_name}]({self.repo_page})'
        self.bot = DingtalkChatbot(conf.webhook, conf.secret)

    def notify(self):
        logging.info(f'Preparing notification: {self.payload}')
        if 'pull_request' in self.payload:
            self._notify_pull_request()
        elif 'head_commit' in self.payload:
            self._notify_push()
        elif 'issue' in self.payload:
            self._notify_issue()
        elif 'starred_at' in self.payload:
            self._notify_star()
        elif 'forkee' in self.payload:
            self._notify_fork()
        elif 'discussion' in self.payload:
            self._notify_discussion()

    def _notify_pull_request(self):
        pr = self.payload['pull_request']
        pr_page = pr['html_url']
        pr_number = pr['number']
        pr_title = pr['title']
        pr_body = pr['body'] or ''
        review = self.payload.get('review')
        comment = self.payload.get('comment')
        if review:
            pr_review_page = review['html_url']
            review_body = review['body'] or ''
            self.bot.send_markdown(
                title='Pull Request Review',
                text=f'{self._md_sender} has {self.action} a pull request review {self.action_prep} {self._md_repo}\n\n'
                     f'[#{pr_number} {pr_title}]({pr_review_page})\n\n'
                     f'> {review_body}'
            )
        elif comment:
            comment_page = comment['html_url']
            comment_body = comment['body'] or ''
            self.bot.send_markdown(
                title='Issue Comment',
                text=f'{self._md_sender} has {self.action} a pull request review comment '
                     f'{self.action_prep} {self._md_repo}\n\n'
                     f'[#{pr_number} {pr_title}]({comment_page})\n\n'
                     f'> {comment_body}'
            )
        else:
            self.bot.send_markdown(
                title='Pull Request',
                text=f'{self._md_sender} has {self.action} a pull request {self.action_prep} {self._md_repo}\n\n'
                     f'[#{pr_number} {pr_title}]({pr_page})\n\n'
                     f'> {pr_body}'
            )
