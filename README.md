## Project Repository:
https://github.com/HADDADSOHAIB/study-mentors-api

## Introduction:
Ce projet et le backend d'une plateforme qui fait les réservations des sessions entre les étudiants et les formateurs. Le front est sur le repository [suivant](https://github.com/HADDADSOHAIB/study-mentors).

Dans les paragraphes suivants, vous trouvez l'analyse de quelque parti du code et en même des propositions des améliorations vu que le code est écrit ça fait long et durant cette période j'ai appris beaucoup des choses.

## Authentification:
Pour gérer l'authentification, le standard de Json Web Token (JWT) est utilisé, vous trouvez ci-dessous la fonction qui fait le sign-up dans le contrôleur([pour plus de details](https://github.com/HADDADSOHAIB/study-mentors-api/tree/master/app/controllers/api/v1)):

```
def create
  account_type = params[:account_type]
  if account_type == 'Student'
    @user = Student.new(user_params)
  elsif account_type == 'Teacher'
    teacher_params = user_params
    teacher_params[:schedule] = {}
    teacher_params[:session_type] = 'online'
    @user = Teacher.new(teacher_params)
  end

  if @user.save
    payload = { user_id: @user.id, account_type: account_type }
    session = JWTSessions::Session.new(payload: payload, refresh_by_access_allowed: true)
    tokens = session.login
    response.set_cookie(
      JWTSessions.access_cookie,
      value: tokens[:access],
      httponly: true,
      secure: Rails.env.production?
    )

    render json: {
      csrf: tokens[:csrf],
      access: tokens[:access],
      current_user: @user,
      categories: []
    }, status: :created
  else
    render json: @user.errors, status: :unprocessable_entity
  end
end
```

Durant la création d'un compte, on reçoit du front le type du compte qu'on veut créer (étudiant ou formateur), puis on crée le token d'authentification et l'envoyé dans la réponse.

--> Améliorations: 

```
  account_type = params[:account_type]
  if account_type == 'Student'
    @user = Student.new(user_params)
  elsif account_type == 'Teacher'
    teacher_params = user_params
    teacher_params[:schedule] = {}
    teacher_params[:session_type] = 'online'
    @user = Teacher.new(teacher_params)
  end
```
Pour améliorer cette partie, je propose de faire la meta programming;
```
 @user = account_type.safe_constantize&.new(eval("#{account_type.downcase}_params"))
```
L'avantage de cette approche c'est qu'on évite beaucoup des ifs/Else, surtout si on ajoute d'autres types de compte, aussi on a passé de 9 lignes de code à 1 ligne.

## Réservations:

Pour faire des réservations, on tape sur le contrôleur [suivant](https://github.com/HADDADSOHAIB/study-mentors-api/blob/master/app/controllers/api/v1/bookings_controller.rb):
```
def create
  teacher = Teacher.find(create_params[:teacher_id])
  student = Student.find(create_params[:student_id])
  category = Category.find(create_params[:category_id])

  @booking = Booking.new(
    teacher: teacher,
    student: student,
    category: category,
    session_type: create_params[:type],
    date: create_params[:from],
    from: create_params[:from],
    to: create_params[:to]
  )

  if @booking.save
    render json: { booking: @booking }, status: :created
  else
    render json: @booking.errors, status: :unprocessable_entity
  end
end
```
Cette fonction permet de créer des réservations.

--> Améliorations:
```
teacher = Teacher.find(create_params[:teacher_id])
student = Student.find(create_params[:student_id])
category = Category.find(create_params[:category_id])
```
Il y un grand problème de sécurité sur cette partie, on ne vérifie pas si le teacher id et student id est bien l'id des personnes qui sont connecté, alors on peut passer des réservations pour d'autre personne.
Ce qu'on doit faire:
1- ajouter la notion de qui est faite la réservation.
2- comparer l'idée de cette personne avec la id dans le payload.

Dans le [contrôleur globale](https://github.com/HADDADSOHAIB/study-mentors-api/blob/master/app/controllers/application_controller.rb), on a ça:
```
 def current_user
    @current_user ||= if payload['account_type'] == 'Teacher'
                        Teacher.find(payload['user_id'])
                      else
                        Student.find(payload['user_id'])
                      end
  end
```
On peut vérifier qui est l'utilisateur connecté avec la variable @current_user.
Ce code aussi peut être amélioré, en effet, nous avons une mauvaise gestion des exceptions, cette fonction:
```
Teacher.find(payload['user_id'])
```
lance une exception s'il ne trouve pas de record dans la base de données, la chose qui peuvent casser le serveur, pour corriger ça, on peut travailler avec:
```
Teacher.find_by(id: payload['user_id'])
```
qui fait la même chose, mais envoyer la valeur nil, s'il le record n'existe pas.