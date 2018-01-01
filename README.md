# DNetORM4.0

Instructions

http://www.cnblogs.com/DNetORM/p/8000373.html

first define the entity

    public class Book
    {
        [Key(IsAutoGenerated = true)]
        public int? BookID { get; set; }
        public string BookName { get; set; }
        public int? AuthorID { get; set; }

        [NotColumn]
        public string AuthorName { get; set; }
        public double? Price { get; set; }

        public DateTime? PublishDate { get; set; }

    }

    public class Author
    {
        [Key(IsAutoGenerated =true)]
        //[Sequence("emp_sequence")]
        public int? AuthorID { get; set; }
        public string AuthorName { get; set; }
        public int? Age { get; set; }
        public bool? IsValid { get; set; }
    }
    
1.add

            using (DNetContext db = new DNetContext())
            {
                var authorid = db.Add(new Author { AuthorName = "jim", Age = 30, IsValid = true });
                db.Add(new Book { BookName = "c#", Price = 20.5, PublishDate = DateTime.Now, AuthorID = authorid });
            }
            
2.delete

            using (DNetContext db = new DNetContext())
            {
                var author = db.GetSingle<Author>(m => true, q => q.OrderBy(m => m.AuthorID));
                var effect = db.Delete(author);

                int authorid = db.GetMax<Author>(m => (int)m.AuthorID);
                db.Delete<Author>(m => m.AuthorID == authorid);

            }

3.update

            using (DNetContext db = new DNetContext())
            {
                var author = db.GetSingle<Author>(m => true, q => q.OrderBy(m => m.AuthorID));
                if (author != null)
                {
                    author.AuthorName = "jim";
                    var effect = db.Update(author);
                }
                db.Update<Author>(m => m.AuthorName = "jim", m => m.AuthorID == 1);
                db.Update<Author>(m => { m.AuthorName = "jim"; m.Age = 30; }, m => m.AuthorID == 1);
                db.Update<Author>(m => new Author { AuthorName = m.AuthorName + "123", IsValid = true }, m => m.AuthorID == 1);
                db.UpdateOnlyFields<Author>(new Author { AuthorName = "123", Age = 20, AuthorID = 1, IsValid = true }, m => new { m.AuthorName, m.Age }, m => m.AuthorID == 1);
                db.UpdateIgnoreFields<Author>(new Author { AuthorName = "123", Age = 20, AuthorID = 1, IsValid = true }, m => m.AuthorName, m => m.AuthorID == 1);
            }
            
4.query (single table)

            using (DNetContext db = new DNetContext())
            {
                var author = db.GetSingle<Author>(m => true, q => q.OrderBy(m => m.AuthorID));

                var book = db.GetSingle<Book>(m => ((DateTime)m.PublishDate).ToString("yyyy-MM-dd") == "2017-11-11");

                var authors = db.GetList<Author>(m => string.IsNullOrEmpty(m.AuthorName) && m.IsValid == true);

                List<dynamic> name = db.GetDistinctList<Author>(m => m.AuthorName.StartsWith("jim") && m.IsValid == true, m => m.AuthorName + "aaa");

                List<string> name1 = db.GetDistinctList<Author, string>(m => m.AuthorName.IndexOf("jim") == 2 && m.IsValid == true, m => m.AuthorName);

                var books = db.GetList<Book>(m => SubQuery.GetList<Author>(n => n.AuthorID > 10, n => n.AuthorID).Contains(m.AuthorID));

                books = db.GetList<Book>(m => m.AuthorID >= SubQuery.GetSingle<Author>(n => n.AuthorID == 10, n => n.AuthorID));

                var authorid = db.GetMax<Author>(m => (int)m.AuthorID);

                WhereBuilder<Author> where = new WhereBuilder<Author>();
                where.And(m => m.AuthorName.Contains("jim"));
                where.And(m => m.AuthorID == 3);
                where.Or(m => m.IsValid == true);
                db.GetList<Author>(where.WhereExpression);


                PageFilter page = new PageFilter { PageIndex = 1, PageSize = 10 };
                page.And<Author>(m => "jim green".Contains(m.AuthorName));
                page.OrderBy<Author>(q => q.OrderBy(m => m.AuthorName).OrderByDescending(m => m.AuthorID));
                PageDataSource<Author> pageSource = db.GetPage<Author>(page);
            }
 5.query (multi tables)    
 
             using (DNetContext db = new DNetContext())
            {

                var books = db.JoinQuery.LeftJoin<Book, Author>((m, n) => m.AuthorID == n.AuthorID && n.IsValid == true)
                    .Fields<Book, Author>((m, n) => new { BookName = m.BookName + "123", AuthorName = SqlFunctions.Count(n.AuthorName) })
                    .OrderByAsc<Book>(m => m.BookName)
                    .GroupBy<Book, Author>((m, n) => new { m.BookName, n.AuthorName })
                    .Where<Book, Author>((m, n) => m.Price > 10 && n.IsValid == true && SubQuery.GetList<Author>(n1 => n1.AuthorID >= 1, n1 => n1.AuthorID).Contains(m.AuthorID))
                    .GetList<Book>();



                var join = db.JoinQueryAlias.LeftJoin<Book, Author>((m, n) => m.AuthorID == n.AuthorID && n.IsValid == true)
                    .InnerJoin<Book, Author>((m1, n) => m1.AuthorID == n.AuthorID && n.IsValid == true)
                    .Fields<Book, Author>((m1, n) => new { AuthorName1 = m1.BookName + n.AuthorName, n })
                    .OrderByAsc<Book>(m => m.BookName);
                PageFilter page = new PageFilter { PageIndex = 1, PageSize = 10 };
                join.GetPage<Book>(page);

            }
6.query (sql)

            using (DNetContext db = new DNetContext())
            {
                StringBuilder sql = new StringBuilder();
                List<DbParameter> parameters = new List<DbParameter>();

                sql.AppendFormat(@"SELECT {0},A.AuthorName FROM Book B 
                LEFT JOIN Author A ON A.AuthorID=B.AuthorID WHERE", SqlBuilder.GetSelectAllFields<Book>("B"));
                sql.Append(" B.BookID>@BookID ");
                parameters.Add(db.GetDbParameter("BookID", 1));

                PageFilter pageFilter = new PageFilter { PageIndex = 1, PageSize = 5 };
                pageFilter.OrderText = "B.BookID ASC";
                PageDataSource <Book> books = db.GetPage<Book>(sql.ToString(), pageFilter, parameters.ToArray());

                List<Book> bks = db.GetList<Book>(sql.ToString(), parameters.ToArray());
            }
7.db transaction

            using (DNetContext db = new DNetContext())
            {
                db.DataBase.BeginTransaction();
                try
                {
                    List<Author> authors = new List<Author>();
                    for (int i = 0; i <= 100; i++)
                    {
                        authors.Add(new Author { AuthorName = "jack" + i.ToString(), Age = 20, IsValid = true });
                    }
                    db.Add(authors);
                    db.DataBase.Commit();
                }
                catch
                {
                    db.DataBase.Rollback();
                }
            }
8.distributed transaction

            DNetTransaction transaction = new DNetTransaction();
            transaction.BeginTransaction();
            try
            {
                using (DNetContext db = new DNetContext())
                {
                    db.DataBase.BeginTransaction();

                    List<Author> authors = new List<Author>();
                    for (int i = 0; i <= 100; i++)
                    {
                        authors.Add(new Author { AuthorName = "测试" + i.ToString(), Age = 20, IsValid = true });
                    }
                    db.Add(authors);
                    transaction.Commit();
                }
            }
            catch
            {
                transaction.Rollback();
            }
if you have some advice please send email to 307474178@qq.com
