Use blueprints to architect a web application
===================================

Author:201932110101李采奕,201932110101窦凯雯,201932110109徐阳奕,201932110108谢思媛

Abstract
--------
A former student developed a web photo album called Photo String for storing photos. He used Flask micro-framework. Photo String allows us to upload a picture and add a description for that picture.

Introduction
--------
·Download the source code of Photo String and run it.
·Make the following blueprints: upload bp, show bp, search bp, and api bp.
·Register the above blueprints to the web application.
·The upload bp blueprint allows uploading a new photo. The associated route is /upload.
·The show bp blueprint allows displaying all photos and their descriptions in chronological order. The associated route is /show.
·The search bp blueprint allows filtering photos according to their descriptions. The associated route is /search/query-string. Only the photos whose descriptions match query-string will be returned as the search result.
·The api bp blueprint allows us to get all photo information in JSON format from command-line.HTTPie is a useful API testing tool. The associated route is /api/json. The returned json string must contain photo ID, date of upload, photo size (in KB) and photo description for each photo.

Code
--------
1.upload.py： 
::

   from flask import Blueprint

   upload_bp = Blueprint('upload', __name__)

   @upload_bp.route('/upload')
   def upload():
       page = '''<form action="/"method="post"enctype="multipart/form-data">
               <input type="file"name="file"><input name="description"><input type="submit"value="Upload"></form>'''
       return page


2.show.py:
::

   from flask import Blueprint
   from service import get_database_photos
   show_bp = Blueprint('show', __name__)


   @show_bp.route('/show')
   def show():
       page = get_database_photos()
       return page

3.search.py:
::

   from flask import Blueprint, request

   from service import get_description_photos

   search_bp = Blueprint('search', __name__)


   @search_bp.route('/search', methods=['POST', 'GET'])
   def search():
       page = '''<form action="/search/query-string"method="post"enctype="multipart/form-data">
                   <input name="description"><input type="submit"value="search"></form>'''
       return page


   @search_bp.route('/search/query-string', methods=['POST', 'GET'])
   def query_string():
       if request.method == 'POST':
           description = request.form['description']
           page = get_description_photos(description)

       return page


4.api.py:
::

   import json
   import os.path

   from flask import Blueprint
   from UseSqlite import RiskQuery

   api_bp = Blueprint('api', __name__)


   @api_bp.route('/api', methods=['POST', 'GET'])
   def api_json():
       rq = RiskQuery('./static/RiskDB.db')
       rq.instructions("SELECT * FROM photo ORDER By time desc")
       rq.do()
       lst = []
       page = ''
       i = 1
       for r in rq.format_results().split('\n\n'):
           photo = r.split(',')
           picture_time = photo[0]
           picture_description = photo[1]
           picture_path = photo[2].strip()
           photo_size = str(format((os.path.getsize(picture_path) / 1024), '.2f')) + 'KB'
           lst = [{'ID': i, 'upload_date': picture_time, 'description': picture_description, 'photo_size': photo_size}]
           lst2 = json.dumps(lst[0], sort_keys=True, indent=4, separators=(',', ':'))
           page += '%s' % lst2
           i += 1
       return page

5.Lab.py:
::

   # -*- coding: utf-8 -*-
   """
   Created on Mon Jun  3 15:42:51 2019

   @author: Administrator
   """

   from flask import Flask, request
   from UseSqlite import InsertQuery
   from datetime import datetime

   from service import get_database_photos
   from upload import upload_bp
   from show import show_bp
   from search import search_bp
   from api import api_bp

   app = Flask(__name__)

   @app.route('/', methods=['POST', 'GET'])
   def main():
       if request.method == 'POST':
           uploaded_file = request.files['file']
           time_str = datetime.now().strftime('%Y%m%d%H%M%S')
           new_filename = time_str + '.jpg'
           uploaded_file.save('./static/upload/' + new_filename)
           time_info = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
           description = request.form['description']
           path = './static/upload/' + new_filename
           iq = InsertQuery('./static/RiskDB.db')
           iq.instructions("INSERT INTO photo Values('%s','%s','%s','%s')" % (time_info, description, path, new_filename))
           iq.do()
           return '<p>You have uploaded %s.<br/> <a href="/">Return</a>.' % (uploaded_file.filename)
       else:
           page = '''
               <a href='/upload'>upload</a>
               <a href='/search'>search</a>
               <a href='/show'>show</a>
               <a href='/api'>api</a>
           '''
           page += get_database_photos()
           return page

   app.register_blueprint(upload_bp)
   app.register_blueprint(show_bp)
   app.register_blueprint(search_bp)
   app.register_blueprint(api_bp)

   if __name__ == '__main__':
       app.run(debug=True)

6.service.py:
::

   from PIL import Image

   from UseSqlite import RiskQuery


   def make_html_paragraph(s):
       if s.strip() == '':
           return ''
       lst = s.split(',')
       picture_path = lst[2].strip()
       picture_name = lst[3].strip()
       im = Image.open(picture_path)
       im.thumbnail((400, 300))
       im.save('./static/figure/' + picture_name, 'jpeg')
       result = '<p>'
       result += '<i>%s</i><br/>' % (lst[0])
       result += '<i>%s</i><br/>' % (lst[1])
       result += '<a href="%s"><img src="./static/figure/%s"alt="风景图"></a>' % (picture_path, picture_name)
       return result + '</p>'


   def make_html_photo(s):
       if s.strip() == '':
           return ''
       lst = s.split(',')
       picture_path = lst[2].strip()
       picture_name = lst[3].strip()
       im = Image.open(picture_path)
       im.thumbnail((400, 300))
       real_path = '.' + picture_path
       result = '<p>'
       result += '<i>%s</i><br/>' % (lst[0])
       result += '<i>%s</i><br/>' % (lst[1])
       result += '<a href="%s"><img src="../static/figure/%s"alt="风景图"></a>' % (real_path, picture_name)
       return result + '</p>'


   def get_database_photos():
       rq = RiskQuery('./static/RiskDB.db')
       rq.instructions("SELECT * FROM photo ORDER By time desc")
       rq.do()
       record = '<p>My past photo</p>'
       for r in rq.format_results().split('\n\n'):
           record += '%s' % (make_html_paragraph(r))
       return record + '</table>\n'


   def get_description_photos(description):
       rq = RiskQuery('./static/RiskDB.db')
       rq.instructions("SELECT * FROM photo where description = '%s' " % description)
       rq.do()
       record = '<p>search result</p>'
       for r in rq.format_results().split('\n\n'):
           record += '%s' % (make_html_photo(r))
       return record + '</table>\n'

7.UseSqlite.py:
::

   # Reference: Dusty Phillips.  Python 3 Objected-oriented Programming Second Edition. Pages 326-328.
   # Copyright (C) 2019 Hui Lan

   import sqlite3

   class Sqlite3Template:
       def __init__(self, db_fname):
           self.db_fname = db_fname

       def connect(self, db_fname):
           self.conn = sqlite3.connect(self.db_fname)

       def instructions(self, query_statement):
           raise NotImplementedError()

       def operate(self):
           self.results = self.conn.execute(self.query) # self.query is to be given in the child classes
           self.conn.commit()

       def format_results(self):
           raise NotImplementedError()  

       def do(self):
           self.connect(self.db_fname)
           self.instructions(self.query)
           self.operate()


   class InsertQuery(Sqlite3Template):
       def instructions(self, query):
           self.query = query


   class RiskQuery(Sqlite3Template):
       def instructions(self, query):
           self.query = query

       def format_results(self):
           output = []
           for row in self.results.fetchall():
               output.append(', '.join([str(i) for i in row]))
           return '\n\n'.join(output)    


   if __name__ == '__main__':

       #iq = InsertQuery('RiskDB.db')
       #iq.instructions("INSERT INTO inspection Values ('FoodSupplies', 'RI2019051301', '2019-05-13', '{}')")
       #iq.do()
       #iq.instructions("INSERT INTO inspection Values ('CarSupplies', 'RI2019051302', '2019-05-13', '{[{\"risk_name\":\"elevator\"}]}')")
       #iq.do()

       rq = RiskQuery('RiskDB.db')
       rq.instructions("SELECT * FROM inspection WHERE inspection_serial_number LIKE 'RI20190513%'")
       rq.do()
       print(rq.format_results())
       
       
References
--------


Read the Docs. https://readthedocs.org/
