## 대용량 처리를 위한 MySQL 이해

### 데이터베이스 병목의 이유

- 스케일 업 : 서버의 성능을 높임
- 스케일 아웃 : 서버의 대수를 늘림

스케일 아웃으로 서버를 관리하기 시작하면서 하나의 데이터 베이스를 공유 → 데이터베이스 병목현상 증가 (다수의 서버, 하나의 데이터베이스)

### 대용량 시스템이 어려운 이유

- 하나의 서버로 감당하기 힘들어 대부분 여러 개의 서버 또는 데이터베이스를 사용함
- 여러 개의 서버에서 유입되는 **데이터의 일관성**을 보장할 수 있어야함
- 코드 한줄이 데이터에 미치는 영향범위가 굉장히 커짐
- 여러 서비스들이 의존을 하고 있어, 시스템 복잡도가 높음

### 대용량 시스템의 특성

- `고가용성` : 언제든 서비스를 이용할 수 있어야함
- `확장성` : 증가하는 데이터와 트래픽에 대응할 수 있어야함
- `관측가능성` : 문제가 생겼을 때 빠르게 인지할 수 있어야하고 문제의 범위를 최소화 할 수 있어야 한다.
- 스케일 아웃, 캐싱, 비동기 큐

## MySQL

### MySQL 아키텍처

- MySQL 엔진
  - 쿼리 파서
    - SQL을 파싱하여 Syntax Tree를 만듬
    - 문법 오류 검사
  - 전처리기
    - 쿼리파서에서 만든 Tree를 바탕으로 전처리 시작
    - 테이블이나 컬럼 존재 여부, 접근권한 등 Semantic 오류 검사
  - 옵티마이저
    - 쿼리를 처리하기 위한 여러 전략을 생성
    - 전략들의 비용정보와 테이블의 통계정보를 이용해 비용을 산정
    - **실행 계획 수립** : 테이블 순서, 불필요한 조건 제거, 통계정보를 바탕으로 전략을 결정
    - 개발자가 힌트를 사용해 도움을 줄 수 있음
  - 쿼리 실행기
    - 옵티마이저가 결정한 실행 계획을 수립하여 스토리지 엔진에 요청
    - Handler API를 이용하여 스토리지 엔진에 요청 전송
- 스토리지 엔진
- 운영체제
- 디스크

## 정규화 반정규화

정규화와 비정규화는 조회와 쓰기 사이의 트레이드 오프가 있음 적절히 사용해야함

`정규화`

- 중복을 제거하고 한곳에서 관리
- 데이터 정합성 유지가 쉬움
- 읽기시 참조 발생

`반정규화`

- 중복을 허용
- 데이터 정합성 유지가 어려움
- 참조없이 읽기 가능

## SNS 서비스를 통한 실습

요구사항

- 회원정보 관리
  - 이메일, 닉네임, 생년월일을 입력받아 저장한다.
  - 닉네임은 10자를 초과할 수 없다.
  - 회원은 닉네임을 변경할 수 있다.
    - 회원의 닉네임 변경이력을 조회할 수 있어야한다.

**MemberDomain**

- `Entity`

  ```java
  package com.example.fastcampusmysql.domain.member.entity;

  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import lombok.Builder;
  import lombok.Getter;
  import org.springframework.util.Assert;

  import java.time.LocalDate;
  import java.time.LocalDateTime;
  import java.util.Objects;

  @Getter
  public class Member {
      private final Long id;
      private String nickname;
      private final String email;
      private final LocalDate birthday;
      private final LocalDateTime createdAt;
      private static final LongNAME_MAX_LENGTH= 10L;

      @Builder
      public Member(Longid, Stringnickname, Stringemail, LocalDatebirthday, LocalDateTimecreatedAt) {
          this.id =id;
          validateNickname(nickname);
          this.nickname = Objects.requireNonNull(nickname);
          this.email = Objects.requireNonNull(email);
          this.birthday = Objects.requireNonNull(birthday);
          this.createdAt =createdAt== null ? LocalDateTime.now() :createdAt;
      }

      public void changeNickname(Stringnickname) {
          Objects.requireNonNull(nickname);
          validateNickname(nickname);
          this.nickname =nickname;
      }

      private void validateNickname(Stringnickname) {
          if (nickname.length() >NAME_MAX_LENGTH) {
              Assert.isTrue(nickname.length() <=NAME_MAX_LENGTH, "최대 길이를 초과했습니다.");
          }
      }
  }

  ```

  ```java
  package com.example.fastcampusmysql.domain.member.entity;

  import lombok.Builder;
  import lombok.Getter;

  import java.time.LocalDateTime;
  import java.util.Objects;

  @Getter
  public class MemberNicknameHistory {

      private final Long id;
      private final Long memberId;
      private final String nickname;
      private final LocalDateTime createdAt;

      @Builder
      public MemberNicknameHistory(Longid, LongmemberId, Stringnickname, LocalDateTimecreatedAt) {
          this.id =id;
          this.memberId = Objects.requireNonNull(memberId);
          this.nickname = Objects.requireNonNull(nickname);
          this.createdAt =createdAt== null ? LocalDateTime.now() :createdAt;
      }
  }

  ```

- `Repository`

  ```java
  package com.example.fastcampusmysql.domain.member.repository;

  import com.example.fastcampusmysql.domain.member.entity.Member;
  import lombok.RequiredArgsConstructor;
  import org.springframework.jdbc.core.RowMapper;
  import org.springframework.jdbc.core.namedparam.BeanPropertySqlParameterSource;
  import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
  import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
  import org.springframework.jdbc.core.namedparam.SqlParameterSource;
  import org.springframework.jdbc.core.simple.SimpleJdbcInsert;
  import org.springframework.stereotype.Repository;

  import java.sql.ResultSet;
  import java.time.LocalDate;
  import java.time.LocalDateTime;
  import java.util.List;
  import java.util.Optional;

  @RequiredArgsConstructor
  @Repository
  public class MemberRepository {

      private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;

      private static final StringTABLE= "member";

      private static final RowMapper<Member>rowMapper= (ResultSetrs, introwNum) -> Member.builder()
              .id(rs.getLong("id"))
              .email(rs.getNString("email"))
              .nickname(rs.getNString("nickname"))
              .birthday(rs.getObject("birthday", LocalDate.class))
              .createdAt(rs.getObject("createdAt", LocalDateTime.class))
              .build();

      public Optional<Member> findById(Longid) {
          Stringsql= String.format("SELECT * FROM %s WHERE id = :id",TABLE);
          SqlParameterSourceparams= new MapSqlParameterSource()
                  .addValue("id",id);
          Membermember= namedParameterJdbcTemplate.queryForObject(sql,params,rowMapper);
          return Optional.of(member);
      }

      public List<Member> findAllByIdIn(List<Long>ids) {
          if (ids.isEmpty())
              return List.of();

          Stringsql= String.format("SELECT * FROM %s WHERE id in (:ids)",TABLE);
          MapSqlParameterSourceparams= new MapSqlParameterSource().addValue("ids",ids);
          return namedParameterJdbcTemplate.query(sql,params,rowMapper);
      }

      public Member save(Membermember) {
  /**
           * member id를 보고 갱신 또는 삽입 정함
  *반환값은Optional<Member>-> member의id를 담아서 반환
  */
  if (member.getId() == null)
              return insert(member);
          return update(member);
      }

      private Member insert(Membermember) {
          SimpleJdbcInsertsimpleJdbcInsert= new SimpleJdbcInsert(namedParameterJdbcTemplate.getJdbcTemplate())
                  .withTableName(TABLE)
                  .usingGeneratedKeyColumns("id")
                  .usingColumns("email", "nickname", "birthday", "createdAt");
          SqlParameterSourceparams= new BeanPropertySqlParameterSource(member);
          varid=simpleJdbcInsert.executeAndReturnKey(params).longValue();

          MemberreturnMember= Member.builder()
                  .id(id)
                  .email(member.getEmail())
                  .nickname(member.getNickname())
                  .birthday(member.getBirthday())
                  .createdAt(member.getCreatedAt())
                  .build();
          returnreturnMember;
      }

      private Member update(Membermember) {
          Stringsql= String.format("UPDATE %s set " +
                  "email=:email, " +
                  "nickname=:nickname, " +
                  "birthday=:birthday " +
                  "WHERE id=:id",TABLE);
          BeanPropertySqlParameterSourceparams= new BeanPropertySqlParameterSource(member);
          namedParameterJdbcTemplate.update(sql,params);
          returnmember;
      }
  }

  ```

  - `NamedParameterJdbcTemplate` : param을 이름으로 맵핑 가능하게 해줌
    - `SqlParameterSource`
      - `MapSqlParameterSource` : Map 기반
      - `BeanPropertySqlParameterSource` : Object 기반
    - `RowMapper` : DB의 응답값을 객체로 변환
  - `SimpleJdbcInsert`
    - `withTableName` : 테이블 이름 지정
    - `usingGeneratedKeyColumns` : 기본키
    - `usingColumns` : 사용할 컬럼 지정
  - `BeanPropertySqlParameterSource` : 객체 중심으로 맵핑
  - `executeAndReturnKey(params)` : DB 쿼리후 생성된 키값을 반환

  ```java
  package com.example.fastcampusmysql.domain.member.repository;

  import com.example.fastcampusmysql.domain.member.entity.MemberNicknameHistory;
  import lombok.RequiredArgsConstructor;
  import org.springframework.jdbc.core.RowMapper;
  import org.springframework.jdbc.core.namedparam.BeanPropertySqlParameterSource;
  import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
  import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
  import org.springframework.jdbc.core.namedparam.SqlParameterSource;
  import org.springframework.jdbc.core.simple.SimpleJdbcInsert;
  import org.springframework.stereotype.Repository;

  import java.sql.ResultSet;
  import java.time.LocalDateTime;
  import java.util.List;
  import java.util.Optional;

  @RequiredArgsConstructor
  @Repository
  public class MemberNicknameHistoryRepository {

      private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;

      private static final String TABLE = "MemberNicknameHistory";

      private static final RowMapper<MemberNicknameHistory> rowMapper = (ResultSet rs, int rowNum) ->
              MemberNicknameHistory.builder()
                      .id(rs.getLong("id"))
                      .memberId(rs.getLong("memberId"))
                      .nickname(rs.getNString("nickname"))
                      .createdAt(rs.getObject("createdAt", LocalDateTime.class))
                      .build();

      public Optional<MemberNicknameHistory> findById(Long id) {
          String sql = String.format("SELECT * FROM %s WHERE id=:id", TABLE);
          SqlParameterSource params = new MapSqlParameterSource()
                  .addValue("id", id);
          MemberNicknameHistory memberNicknameHistory = namedParameterJdbcTemplate.queryForObject(sql, params, rowMapper);
          return Optional.of(memberNicknameHistory);
      }

      public List<MemberNicknameHistory> findAllByMemberId(Long memberId) {
          String sql = String.format("SELECT * FROM %s WHERE memberId=:memberId", TABLE);
          SqlParameterSource params = new MapSqlParameterSource()
                  .addValue("memberId", memberId);
          return namedParameterJdbcTemplate.query(sql, params, rowMapper);
      }

      public MemberNicknameHistory save(MemberNicknameHistory memberNicknameHistory) {
          if (memberNicknameHistory.getId() == null)
              return insert(memberNicknameHistory);
          throw new UnsupportedOperationException("MemberNicknameHistory는 갱신을 지원하지 않습니다.");
      }

      private MemberNicknameHistory insert(MemberNicknameHistory memberNicknameHistory) {
          SimpleJdbcInsert simpleJdbcInsert = new SimpleJdbcInsert(namedParameterJdbcTemplate.getJdbcTemplate())
                  .withTableName(TABLE)
                  .usingGeneratedKeyColumns("id")
                  .usingColumns("memberId", "nickname", "createdAt");

          SqlParameterSource params = new BeanPropertySqlParameterSource(memberNicknameHistory);

          long id = simpleJdbcInsert.executeAndReturnKey(params).longValue();

          return MemberNicknameHistory.builder()
                  .id(id)
                  .memberId(memberNicknameHistory.getMemberId())
                  .nickname(memberNicknameHistory.getNickname())
                  .createdAt(memberNicknameHistory.getCreatedAt())
                  .build();
      }
  }
  ```

- `Service`

  ```java
  package com.example.fastcampusmysql.domain.member.service;

  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import com.example.fastcampusmysql.domain.member.dto.MemberNicknameHistoryDto;
  import com.example.fastcampusmysql.domain.member.entity.Member;
  import com.example.fastcampusmysql.domain.member.mapper.MemberDomainDtoMapper;
  import com.example.fastcampusmysql.domain.member.repository.MemberNicknameHistoryRepository;
  import com.example.fastcampusmysql.domain.member.repository.MemberRepository;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;

  import java.util.List;
  import java.util.stream.Collectors;

  @RequiredArgsConstructor
  @Service
  public class MemberReadService {

      private final MemberRepository memberRepository;
      private final MemberNicknameHistoryRepository memberNicknameHistoryRepository;

      public MemberDto getMember(Long memberId) {
          var returnMember = memberRepository.findById(memberId).orElseThrow();
          return MemberDomainDtoMapper.toDto(returnMember);
      }

      public List<MemberDto> getMembers(List<Long> ids) {
          return memberRepository.findAllByIdIn(ids)
                  .stream()
                  .map(m -> MemberDomainDtoMapper.toDto(m))
                  .collect(Collectors.toList());
      }

      public List<MemberNicknameHistoryDto> getNicknameHistory(Long memberId) {
          return memberNicknameHistoryRepository.findAllByMemberId(memberId)
                  .stream()
                  .map(h -> MemberDomainDtoMapper.toDto(h))
                  .collect(Collectors.toList());
      }
  }
  ```

  ```java
  package com.example.fastcampusmysql.domain.member.service;

  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import com.example.fastcampusmysql.domain.member.dto.RegisterMemberCommand;
  import com.example.fastcampusmysql.domain.member.entity.Member;
  import com.example.fastcampusmysql.domain.member.entity.MemberNicknameHistory;
  import com.example.fastcampusmysql.domain.member.mapper.MemberDomainDtoMapper;
  import com.example.fastcampusmysql.domain.member.repository.MemberNicknameHistoryRepository;
  import com.example.fastcampusmysql.domain.member.repository.MemberRepository;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;

  @RequiredArgsConstructor
  @Service
  public class MemberWriteService {

      private final MemberRepository memberRepository;
      private final MemberNicknameHistoryRepository memberNicknameHistoryRepository;

      // 회원 정보 등록
      public MemberDto register(RegisterMemberCommand command) {
           var member = Member.builder()
                   .nickname(command.nickname())
                   .email(command.email())
                   .birthday(command.birthday())
                   .build();
           var saveMember = memberRepository.save(member);

          saveMemberNicknameHistory(saveMember);

          return MemberDomainDtoMapper.toDto(saveMember);
      }

      /**
       * 1. 회원의 이름 변경
       * 2. 변경 내역 저장
       */
      public void changeNickname(Long memberId, String nickname) {

          // 1. 회원 이름 변경
          Member member = memberRepository.findById(memberId).orElseThrow();
          member.changeNickname(nickname);
          memberRepository.save(member);

          // 2. 변경 내역 저장
          saveMemberNicknameHistory(member);
      }

      private void saveMemberNicknameHistory(Member member) {
          MemberNicknameHistory memberNicknameHistory = MemberNicknameHistory.builder()
                  .memberId(member.getId())
                  .nickname(member.getNickname())
                  .build();
          memberNicknameHistoryRepository.save(memberNicknameHistory);
      }
  }
  ```

- `Controller`

  ```java
  package com.example.fastcampusmysql.application.controller;

  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import com.example.fastcampusmysql.domain.member.dto.MemberNicknameHistoryDto;
  import com.example.fastcampusmysql.domain.member.dto.RegisterMemberCommand;
  import com.example.fastcampusmysql.domain.member.service.MemberReadService;
  import com.example.fastcampusmysql.domain.member.service.MemberWriteService;
  import lombok.RequiredArgsConstructor;
  import org.springframework.web.bind.annotation.*;

  import java.util.List;

  @RequiredArgsConstructor
  @RestController
  @RequestMapping("/members")
  public class MemberController {

      private final MemberReadService memberReadService;
      private final MemberWriteService memberWriteService;

      @GetMapping("/{id}")
      public MemberDto getMember(@PathVariable("id") Long id) {
          return memberReadService.getMember(id);
      }

      @GetMapping("/{memberId}/history")
      public List<MemberNicknameHistoryDto> getMemberNicknameHitory(@PathVariable("memberId") Long memberId) {
          return memberReadService.getNicknameHistory(memberId);
      }

      @PostMapping
      public MemberDto register(@RequestBody RegisterMemberCommand command) {
          return memberWriteService.register(command);
      }

      @PutMapping("/{id}")
      public MemberDto changeNickname(@PathVariable("id") Long id, @RequestBody String nickname) {
          memberWriteService.changeNickname(id, nickname);
          return memberReadService.getMember(id);
      }
  }
  ```

`FollowDomain`

- `Entity`

  ```java
  package com.example.fastcampusmysql.domain.follow.entity;

  import lombok.Builder;
  import lombok.Getter;

  import java.time.LocalDateTime;
  import java.util.Objects;

  @Getter
  public class Follow {

      private Long id;
      private Long fromMemberId;
      private Long toMemberId;
      private LocalDateTime createdAt;

      @Builder
      public Follow(Long id, Long fromMemberId, Long toMemberId, LocalDateTime createdAt) {
          this.id = id;
          this.fromMemberId = Objects.requireNonNull(fromMemberId);
          this.toMemberId = Objects.requireNonNull(toMemberId);
          this.createdAt = createdAt == null ? LocalDateTime.now() : createdAt;
      }
  }
  ```

- `Repository`

  ```java
  package com.example.fastcampusmysql.domain.follow.repository;

  import com.example.fastcampusmysql.domain.follow.entity.Follow;
  import lombok.RequiredArgsConstructor;
  import org.springframework.jdbc.core.RowMapper;
  import org.springframework.jdbc.core.namedparam.BeanPropertySqlParameterSource;
  import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
  import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
  import org.springframework.jdbc.core.namedparam.SqlParameterSource;
  import org.springframework.jdbc.core.simple.SimpleJdbcInsert;
  import org.springframework.stereotype.Repository;

  import java.sql.ResultSet;
  import java.time.LocalDateTime;
  import java.util.List;

  @RequiredArgsConstructor
  @Repository
  public class FollowRepository {

      private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;

      private static final String TABLE = "Follow";

      private static final RowMapper<Follow> rowMapper = (ResultSet rs, int rowNum) -> Follow.builder()
              .id(rs.getLong("id"))
              .fromMemberId(rs.getLong("fromMemberId"))
              .toMemberId(rs.getLong("toMemberId"))
              .createdAt(rs.getObject("createdAt", LocalDateTime.class))
              .build();

      public List<Follow> findAllByFromMemberId(Long fromMemberId) {
          String sql = String.format("SELECT * FROM %s WHERE fromMemberId=:fromMemberId", TABLE);
          MapSqlParameterSource params = new MapSqlParameterSource()
                  .addValue("fromMemberId", fromMemberId);
          return namedParameterJdbcTemplate.query(sql, params, rowMapper);
      }

      public Follow save(Follow follow) {
          if (follow.getId() == null)
              return insert(follow);
          throw new UnsupportedOperationException("Follow는 갱신을 지원하지 않습니다.");
      }

      private Follow insert(Follow follow) {
          SimpleJdbcInsert simpleJdbcInsert = new SimpleJdbcInsert(namedParameterJdbcTemplate.getJdbcTemplate())
                  .withTableName(TABLE)
                  .usingGeneratedKeyColumns("id")
                  .usingColumns("fromMemberId", "toMemberId", "createdAt");
          SqlParameterSource params = new BeanPropertySqlParameterSource(follow);

          long id = simpleJdbcInsert.executeAndReturnKey(params).longValue();
          return Follow.builder()
                  .id(id)
                  .fromMemberId(follow.getFromMemberId())
                  .toMemberId(follow.getToMemberId())
                  .createdAt(follow.getCreatedAt())
                  .build();
      }
  }
  ```

- `Service`

  ```java
  package com.example.fastcampusmysql.domain.follow.service;

  import com.example.fastcampusmysql.domain.follow.followDto.FollowDto;
  import com.example.fastcampusmysql.domain.follow.mapper.FollowDomainMapper;
  import com.example.fastcampusmysql.domain.follow.repository.FollowRepository;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;

  import java.util.List;
  import java.util.stream.Collectors;

  @RequiredArgsConstructor
  @Service
  public class FollowReadService {

      private final FollowRepository followRepository;

      public List<FollowDto> getFollowings(Long fromMemberId) {
          return followRepository.findAllByFromMemberId(fromMemberId).stream()
                  .map(f -> FollowDomainMapper.toDto(f))
                  .collect(Collectors.toList());
      }
  }
  ```

  ```java
  package com.example.fastcampusmysql.domain.follow.service;

  import com.example.fastcampusmysql.domain.follow.entity.Follow;
  import com.example.fastcampusmysql.domain.follow.repository.FollowRepository;
  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;
  import org.springframework.util.Assert;

  @RequiredArgsConstructor
  @Service
  public class FollowWriteService {

      private final FollowRepository followRepository;

      public void create(MemberDto fromMember, MemberDto toMember) {
          Assert.isTrue(!fromMember.id().equals(toMember.id()), "From, To 회원이 동일합니다.");

          Follow follow = Follow.builder()
                  .fromMemberId(fromMember.id())
                  .toMemberId(toMember.id())
                  .build();

          followRepository.save(follow);
      }
  }
  ```

- `Controller`

  ```java
  package com.example.fastcampusmysql.application.controller;

  import com.example.fastcampusmysql.application.usacase.CreateFollowMemberUsacase;
  import com.example.fastcampusmysql.application.usacase.GetFollowingMembersUsacase;
  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import lombok.RequiredArgsConstructor;
  import org.springframework.web.bind.annotation.*;

  import java.util.List;

  @RequiredArgsConstructor
  @RestController
  @RequestMapping("/follow")
  public class FollowController {

      private final CreateFollowMemberUsacase createFollowMemberUsacase;
      private final GetFollowingMembersUsacase followingMembersUsacase;

      @PostMapping("/{fromId}/{toId}")
      public void create(@PathVariable("fromId") Long fromId, @PathVariable("toId") Long toId) {
          createFollowMemberUsacase.execute(fromId, toId);
      }

      @PostMapping("/members/{fromId}")
      public List<MemberDto> getFollowings(@PathVariable("fromId") Long fromId) {
          return followingMembersUsacase.execute(fromId);
      }
  }
  ```

- `Usacase`

  ```java
  package com.example.fastcampusmysql.application.usacase;

  import com.example.fastcampusmysql.domain.follow.service.FollowWriteService;
  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import com.example.fastcampusmysql.domain.member.service.MemberReadService;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;

  @RequiredArgsConstructor
  @Service
  public class CreateFollowMemberUsacase {

      private final MemberReadService memberReadService;
      private final FollowWriteService followWriteService;
      /**
       * 1. 입력받는 memberId로 회원조회
       * 2. FollowWriteService.create()
       */
      public void execute(Long fromMemberId, Long toMemberId) {
          MemberDto fromMember = memberReadService.getMember(fromMemberId);
          MemberDto toMember = memberReadService.getMember(toMemberId);
          followWriteService.create(fromMember, toMember);
      }
  }
  ```

  ```java
  package com.example.fastcampusmysql.application.usacase;

  import com.example.fastcampusmysql.domain.follow.followDto.FollowDto;
  import com.example.fastcampusmysql.domain.follow.service.FollowReadService;
  import com.example.fastcampusmysql.domain.member.dto.MemberDto;
  import com.example.fastcampusmysql.domain.member.service.MemberReadService;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;

  import java.util.List;
  import java.util.stream.Collectors;

  @RequiredArgsConstructor
  @Service
  public class GetFollowingMembersUsacase {

      private final MemberReadService memberReadService;
      private final FollowReadService followReadService;

      public List<MemberDto> execute(Long fromId) {
          List<FollowDto> followings = followReadService.getFollowings(fromId);
          List<Long> followingsMemberIds = followings.stream()
                  .map(f -> f.id())
                  .collect(Collectors.toList());
          return memberReadService.getMembers(followingsMemberIds);
      }
  }
  ```

- `MemberTest`

  ```java
  package com.example.fastcampusmysql.domain.util;

  import com.example.fastcampusmysql.domain.member.entity.Member;
  import org.jeasy.random.EasyRandom;
  import org.jeasy.random.EasyRandomParameters;

  public class MemberFixtureFactory {

      public static Member create() {
          var param = new EasyRandomParameters();
          return new EasyRandom(param).nextObject(Member.class);
      }

      public static Member create(Long seed) {
          var param = new EasyRandomParameters().seed(seed);
          return new EasyRandom(param).nextObject(Member.class);
      }
  }
  ```

  ```java
  package com.example.fastcampusmysql.domain.util;

  import com.example.fastcampusmysql.domain.member.entity.Member;
  import org.jeasy.random.EasyRandom;
  import org.jeasy.random.EasyRandomParameters;

  public class MemberFixtureFactory {

      public static Member create() {
          var param = new EasyRandomParameters();
          return new EasyRandom(param).nextObject(Member.class);
      }

      public static Member create(Long seed) {
          var param = new EasyRandomParameters().seed(seed);
          return new EasyRandom(param).nextObject(Member.class);
      }
  }
  ```

> 도메인 간의 제어는 여러가지 아키텍처가 존재
> ex) 헥사고날 아키텍처, DDD 아키텍처, 레이어드 아키텍처, usacase 아키텍처, 퍼사드 아키텍처

> 정규화 O → 데이터의 최신성을 보장 O
> 정규화 X → 데이터의 최신성을 보장 X ex) history

### 실무에서 정규화, 비정규화

- 정규화시 고려할 점
  - 데이터의 최신성을 보장?
  - 히스토리성 데이터는 정규화 X
  - 데이터 변경 주기와 조회 주기는 어떻게 되나?
  - 객체(테이블) 탐색 깊이가 얼마나 깊은가
- 정규화를 하기로 했다면 읽기시 데이터를 어떻게 가져올 것인가
  - 테이블 조인
    - 테이블 조인은 서로 다른 테이블의 결합도를 엄청나게 높인다
    - 조회시 성능이 좋은 별도 데이터베이스나 캐싱등 다양한 최적화 기법을 이용할 수 있음
    - 조인을 사용하게 되면, 이런 기법들을 사용하는데 제한이 있거나 더 많은 리소스가 들 수 있다.
    - 읽기 쿼리 한번 더 발생되는 것을 그렇게 큰 부담이 아닐 수도 있음
